---
title: "Envoy Hot Restart Implementation Analysis"
date: 2020-07-18T20:29:23+08:00
draft: false
---

## Introduction

Envoy 热重启功能的目的是为了减少 Envoy 重启/升级所需的 downtime。当一个新的 Envoy 进程启动时如果有在运行的 Envoy 进程，新的 Envoy 就会执行热重启逻辑。
热重启的实现主要依赖 SHM(Linux Shared Memory), UDS(Unix Domain Socket) 和 Protobuf。SHM 存储了热重启的基本状态，和用于进程间同步的锁。Envoy 启动后会首先检测 SHM 中的状态，如果存在错误则立即退出。UDS 用于 child 和 server 之间的通信，消息以 Protobuf 格式序列化，消息类型包括从 parent 获取 listen sockets, 从 parent 获取 stats。 Envoy 热重启会保证 listen sockets 和 stats 在进程间无中断转移，而 parent 上已经建立的连接是会断开的。如果 7 层协议具有优雅断开机制，则流量无损。如果像是 tcp_proxy 这样的连接则是可能有损流量的。

## SHM

`struct SharedMemory` 是 SHM 对应的数据结构，`size_` 是结构体大小, `version_` 为热重启协议版本，只有版本相等进程之间才能进行热重启。`log_lock` 和 `access_log_lock` 用于控制进程间日志操作的同步，这里可以先忽略。`flags_` 存储状态标志，目前只有 `SHMEM_FLAGS_INITIALIZING`，表示 Envoy 是否处于初始化阶段。

``` c++
// Increment this whenever there is a shared memory / RPC change that will prevent a hot restart
// from working. Operations code can then cope with this and do a full restart.
const uint64_t HOT_RESTART_VERSION = 11;

/**
 * Shared memory segment. This structure is laid directly into shared memory and is used amongst
 * all running envoy processes.
 */
struct SharedMemory {
  uint64_t size_;
  uint64_t version_;
  pthread_mutex_t log_lock_;
  pthread_mutex_t access_log_lock_;
  std::atomic<uint64_t> flags_;
};
static const uint64_t SHMEM_FLAGS_INITIALIZING = 0x1;
```

## RPC

Envoy 基于 UDS 和 Protobuf 实现了一套简单的 RPC，专门用于热重启。Envoy 启动时会绑定两个 UDS，分别作为 child 和 parent。下面是 `cat /proc/<envoy pid>/net/unix` 的结果:

``` bash
istio-proxy@newsrec-engine-docker-test-ab-3-8865k-mjrm0-6c79c45cb4-8sz26:/$ cat /proc/31/net/unix
Num       RefCount Protocol Flags    Type St Inode Path
000000006f0b8f0a: 00000002 00000000 00000000 0002 01 1895644675 @envoy_domain_socket_parent_0@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
00000000462982b6: 00000002 00000000 00000000 0002 01 1895644674 @envoy_domain_socket_child_0@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

从上面可以看到 Envoy 进程绑定了 `envoy_domain_socket_child_0, envoy_domain_socket_parent_0` 这两个 UDS，其命名格式为 `envoy_domain_socket_<role>_<id>` role 为 parent/child, id 是一个基于 epoch 计算的值。当前 Envoy 进程用 epoch -1 可以得到 parent 的 UDS, 用 epoch + 1 即可得到 child 的 UDS。

接下来看一看 RPC 消息定义, 如下所示, 是 Protobuf 定义:

``` protobuf
message HotRestartMessage {
  // Child->parent requests
  message Request {
    message PassListenSocket {
      string address = 1;
    }
    message ShutdownAdmin {
    }
    message Stats {
    }
    message DrainListeners {
    }
    message Terminate {
    }
    oneof request {
      PassListenSocket pass_listen_socket = 1;
      ShutdownAdmin shutdown_admin = 2;
      Stats stats = 3;
      DrainListeners drain_listeners = 4;
      Terminate terminate = 5;
    }
  }

  // Parent->child replies
  message Reply {
    message PassListenSocket {
      int32 fd = 1;
    }
    message ShutdownAdmin {
      uint64 original_start_time_unix_seconds = 1;
    }
    message Span {
      uint32 first = 1;
      uint32 last = 2; // inclusive
    }
    message RepeatedSpan {
      repeated Span spans = 1;
    }
    message Stats {
      // Values for server_stats, which don't fit with the "combination logic" approach.
      uint64 memory_allocated = 1;
      uint64 num_connections = 2;

      // Keys are fully qualified stat names.
      //
      // The amount added to the counter since the last time a message included the counter in this
      // map. (The first time a counter is included in this map, it's the amount added since the
      // final latch() before hot restart began).
      map<string, uint64> counter_deltas = 3;
      // The parent's current values for various gauges in its stats store.
      map<string, uint64> gauges = 4;
      // Maps the string representation of a StatName into an array of Spans,
      // which indicate which of the StatName tokens are dynamic. For example,
      // if we are recording a counter or gauge named "a.b.c.d.e.f", where "a",
      // and "d.e" were created from a StatNameDynamicPool, then we'd map
      // "a.b.c.d.e.f" to the span array [[0,0], [3,4]], where the [0,0] span
      // covers the "a", and the [3,4] span covers "d.e".
      map<string, RepeatedSpan> dynamics = 5;
    }
    oneof reply {
      // When this oneof is of the PassListenSocketReply type, there is a special
      // implied meaning: the recvmsg that got this proto has control data to make
      // the passing of the fd work, so make use of CMSG_SPACE etc.
      PassListenSocket pass_listen_socket = 1;
      ShutdownAdmin shutdown_admin = 2;
      Stats stats = 3;
    }
  }

  oneof requestreply {
    Request request = 1;
    Reply reply = 2;
  }

  bool didnt_recognize_your_last_message = 3;
}
```

目前 RPC 消息都是由 child 主动发送给 parent, parent 发送回复之后, child 再发送下一个请求。 请求的类型有 `PassListenSocket`, `ShutdownAdmin`, `Stats`, `DrainListeners`,  `Terminate`. `PassListenSocket` 用于 child 向 parent 请求传递 listen socket。`ShutdownAdmin` 用于 child 请求 parent 停止监听 admin port, 然后 child 开始监听 admin port。`Stats` 用于传递监控指标。`DrainListeners` 用于 child 初始化完所有 workers，已经做好接管流量的准备时调用，收到这个请求后 parent 会关闭 所有 listener, 停止接收流量。`Terminate` 用于在 graceful 时间之后，让 parent 结束，parent 在收到这个消息后会立即结束。

## Listen FD 传递

Unix domain socket 支持在调用 `sendmsg` 传入类型为 `SCM_RIGHTS` 的 [ancillary data](https://man7.org/linux/man-pages/man7/unix.7.html), 这一机制被用来[传递 file descriptor](https://copyconstruct.medium.com/file-descriptor-transfer-over-unix-domain-sockets-dcbbf5b3b6ec)。

Envoy `void HotRestartingBase::sendHotRestartMessage(sockaddr_un& address,
                                              const HotRestartMessage& proto)` [源码](https://github.com/envoyproxy/envoy/blob/v1.17.1/source/server/hot_restarting_base.cc#L96)

```cpp
void HotRestartingBase::sendHotRestartMessage(sockaddr_un& address,
                                              const HotRestartMessage& proto) {
  // Child 初始化 Listener, 发现 Listener 的 reuse_port 属性为 false, 则从 parent 获取 listen socket 的 fd 

  // ...

  // send_buf 是 proto message 序列化后的数据, 可能会分成多个 UDP message 来发送

  uint8_t* next_byte_to_send = send_buf.data();
  uint64_t sent = 0;
  while (sent < total_size) {
    const uint64_t cur_chunk_size = std::min(MaxSendmsgSize, total_size - sent);
    iovec iov[1];
    iov[0].iov_base = next_byte_to_send;
    iov[0].iov_len = cur_chunk_size;
    next_byte_to_send += cur_chunk_size;
    sent += cur_chunk_size;
    msghdr message;
    memset(&message, 0, sizeof(message));
    message.msg_name = &address;
    message.msg_namelen = sizeof(address);
    message.msg_iov = iov;
    message.msg_iovlen = 1;

    // 如果是 kPassListenSocket 的消息，则传递 fd

    // Control data stuff, only relevant for the fd passing done with PassListenSocketReply.
    uint8_t control_buffer[CMSG_SPACE(sizeof(int))];
    if (replyIsExpectedType(&proto, HotRestartMessage::Reply::kPassListenSocket) &&
        proto.reply().pass_listen_socket().fd() != -1) {
      memset(control_buffer, 0, CMSG_SPACE(sizeof(int)));
      message.msg_control = control_buffer;
      message.msg_controllen = CMSG_SPACE(sizeof(int));
      cmsghdr* control_message = CMSG_FIRSTHDR(&message);
      control_message->cmsg_level = SOL_SOCKET;
      control_message->cmsg_type = SCM_RIGHTS;
      control_message->cmsg_len = CMSG_LEN(sizeof(int));
      *reinterpret_cast<int*>(CMSG_DATA(control_message)) = proto.reply().pass_listen_socket().fd();
      ASSERT(sent == total_size, "an fd passing message was too long for one sendmsg().");
    }

    sendmsg(my_domain_socket_, &message, 0);
  }

  // ...
}
```

## 时序控制

热重启有较为严格的时序控制。Envoy 通过启动时传入 `--restart-epoch` 来为当前进程分配一个 epoch 用于时序控制，每次分配时 epoch 需要先自增 1。在同一时刻最多能够同时存在 3 个 Envoy 进程。例如这个场景，epoch 1 已经初始化完成，epoch 0 正等待 shutdown,  此时 epoch 2 启动，则会触发 epoch 1 进入热重启过程，epoch 1 同时会立刻调用 `sendParentAdminShutdownRequest()` 结束掉 epoch 0。如果 epoch 2 启动的时候，epoch 1 还未初始化完成，则 epoch 2 会立刻退出。

## 热重启的风险

Envoy 热重启过程中，由于有两个 Envoy 进程同时运行，所以内存占用会倍增，此时有 OOM 的风险。所以在部署运维时，需要对 Envoy 的 rss 指标进行监控，当其大于 memory.limit 的 50% 时告警。同时避免频繁的热重启操作。
