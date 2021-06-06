---
title: "Envoy Tls Tansport Socket"
date: 2021-06-06T20:34:02+08:00
draft: false
---

## Concepts

**TransportSocket**

`TransportSocket` 是 Envoy 中为 socket 定义的通用接口，其底层实现可以拓展, `tls` TransportSocket 插件就是利用这一机制在连接建立时处理 tls 相关逻辑。

## Configuration

Envoy 可以作为 server 接收下游连接，同时作为 client 发起上游连接。所以 tls TransportSocket 配置也分为 Server 和 Client 配置。

Envoy 提供了 double-proxy 的例子
https://github.com/envoyproxy/envoy/tree/v1.18.3/examples/double-proxy

该例子中部署了一个 app + envoy 和一个 db + envoy, 网络拓扑是

```
[ app --http--> proxy ]   --https-->   [ proxy --http--> db ]
        下游                                  上游
```

下面是截取上游的 proxy 的 Listener 配置:

```yaml
  listeners:
  - name: postgres_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 5432
    listener_filters:
    - name: "envoy.filters.listener.tls_inspector"
      typed_config: {}
    filter_chains:
    - filters:
      - name: envoy.filters.network.postgres_proxy
        # ...
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          require_client_certificate: true
          common_tls_context:
            tls_certificates:
            - certificate_chain:
                filename: certs/servercert.pem
              private_key:
                filename: certs/serverkey.pem
            validation_context:
              match_subject_alt_names:
              - exact: proxy-postgres-frontend.example.com
              trusted_ca:
                filename: certs/cacert.pem
```

tls 配置的部分在`envoy.transport_sockets.tls` ,这里使用的是静态文件配置和一些基础的配置项。

`common_tls_context.tls_certificates` 是一个列表，表示一对服务器证书和私钥，可以为 Envoy 配置多个服务器证书。

`validation_context` 配置可信 CA，用于双向认证的场景，即 `require_client_certificate` 为 true 的场景。

## 基础功能

**SNI 支持**

https://www.envoyproxy.io/docs/envoy/v1.18.3/faq/configuration/sni

需要添加 `envoy.filters.listener.tls_inspector` 插件，然后在 filter chain match 中配置 `server_names` 字段, 如果 client hello 匹配不上，下游连接就会被 reset。

**根据 client tls 参数选择服务器证书**

在收到 client hello 后， Envoy 可以根据 client 端的参数，例如是否支持 ECDSA, 是否支持 ocsp_staple 等情况来自动选择合适的证书。
匹配规则是根据 `tls_certificates` 选择第一个 match 的。

https://github.com/envoyproxy/envoy/blob/v1.18.3/source/extensions/transport_sockets/tls/context_impl.cc#L1094

**客户端证书透传**

即 `XFCC` header 配置。利用该功能，可以用来实现客户端证书认证。

## 进阶功能

### SDS

SDS 是 xDS 中用来传输 tls 相关资源的 gRPC 服务。它可以用来传输服务器证书，私钥，CA 和 tls session tickets。

Istio sds server: https://github.com/istio/istio/blob/1.10.0/security/pkg/nodeagent/sds/server.go

Istio sds server 内嵌在 pilot-agent 上, 通过 Unix Domain Socket 向 Envoy 提供 SDS 服务。

使用 SDS 的好处有: 1. 证书不落盘, 更为安全 2. 证书管理更加方便。

正因为有 SDS server 的存在，排查 tls 问题的链路又多了一环。Envoy 的 `config_dump` 中会显示证书状态，如果处在 `warming` 状态，那么可能需要看下是否是 sds 服务出现了问题。
证书没有 ready 的情况下，Envoy 也会 reset 下游连接。