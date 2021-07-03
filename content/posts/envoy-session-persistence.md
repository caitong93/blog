---
title: "Envoy Session Persistence"
date: 2021-07-02T13:04:28+08:00
draft: true
---

## 什么是会话保持


我们可以从[这篇文章](https://www.haproxy.com/blog/load-balancing-affinity-persistence-sticky-sessions-what-you-need-to-know/)了解到
从上面的文章中截取一些要点:

>The difference between persistence and affinity
>
>**Affinity**: this is when we use an information from a layer below the application layer to maintain a client request to a single server
>
>**Persistence**: this is when we use Application layer information to stick a client to a single server
>
>**sticky session**: a sticky session is a session maintained by persistence

也就是说 affinity 是一种利用应用层以下的将请求映射到一个 server 的方式。而 persistence 是利用应用层的信息让来自一个 client 的请求都"粘"在一个 server 上。Affinity 不保证会话一定能保持，而 persistence 可以。

这段描述里着重强调了**应用层**，我对此的理解是: 会话处在应用层，例如 HTTP session，只有利用应用层的信息，才能判断出是否为同一个会话。应用层以下的信息是做不到的。

日常沟通的时候，我们可能不会区分 Affinity 和 Persistence，统一都用会话保持来表达。但一旦了解了其中的区别，就需要知道例如源 IP 会话保持，说的是 affinity。基于
[tls session id 会话保持](https://www.haproxy.com/fr/blog/maintain-affinity-based-on-ssl-session-id/
)，也是 affinity。基于 cookie 的会话保持，则是 persistence。

Haproxy 的 [stick table](https://www.haproxy.com/blog/introduction-to-haproxy-stick-tables/)，是一个基于内存缓存的模块，它可以实现 affinity 和 persistence。

## 和负载均衡的关系

基于 hash 的负载均衡算法，因为是确定的(deterministic)， 可以做到 affinity。

Envoy 提供了 hash policy 机制，配置在路由上，配合插件可以做到提取请求的任意信息来做 hash。

但 Envoy 缺少会话保持功能，缺少一个前置于负载均衡的会话保持模块。通过这个会话保持模块，请求处理可以 bypass loadbalance。

## Envoy 会话保持功能的现状

https://github.com/envoyproxy/envoy/issues/16698

wbp 同学正在推动 Envoy 会话保持功能的开发。目前已经内部实现了一个基于 HTTP Cookie 的会话保持功能。
我们的场景是用 Envoy 来做 7 层负载均衡。对比成熟的商业化/开源的负载均衡来看，Envoy 在这一块的能力还差得比较多。