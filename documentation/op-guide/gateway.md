# etcd 网关

> 注： 内容翻译自 [etcd gateway](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/gateway.md)

## etcd 网关是什么

etcd 网关是一个简单的 TCP 代理，转发网络数据到 etcd 集群。网关是无状态和透明的;它既不检查客户端请求也不干涉集群应答。

网关支持多 etcd 服务器终端。当网关启动时，它随机的选择一个 etcd 服务器终端并转发所有请求到这个终端。这个终端服务锁偶请求直到网关发现一个网络失败。如果网关检测到终端失败，它将切换到其他的终端，如果可用，从而给它的客户端隐藏失败。其他重试策略，比如带权重的轮询，可能在未来支持。

## 何时使用 etcd 网关

每个访问 etcd 的应用必须首先要有 etcd 集群客户端终端的地址。如果在同一台服务器上的多个应用访问同一个 etcd 集群，每个应用都需要知道 etcd 集群的公告的客户端终端。如果 etcd 集群被重新配置为使用不同的终端，每个应用都需要更新它的终端列表。这个大规模的重新配置是令人生厌的，而且容易出错。

etcd 网关通过以稳定本地终端的方式来解决这个问题。典型 etcd 网关配置是在每台机器上运行网关监听在本地地址，而每个 etcd 应用连接到它本地的网关。结果是仅仅网关需要更新它的终端而不是更新每个应用。

总的来说，为了自动扩散集群终端变化，etcd 网关运行在每台机器，服务于访问同一个 etcd 集群的多个应用。

## 何时不使用 etcd 网关

- 提升性能

网关不是设计用来提升 etcd 集群的性能。它不提供缓存，监听联合或者批量。etcd 团队正在开发缓存代理，设计用于提升集群可扩展性。

- 运行在集群管理系统之上

高级集群管理系统比如 Kubernetes 原生支持服务发现。应用可以访问使用 DNS 名称 或者系统管理的虚拟IP地址来访问 etcd 集群。例如，kube-proxy 等价于 etcd 网关。

## 启动 etcd 网关

假定 etcd 集群有下列静态终端：

|Name|Address|Hostname|
|------|---------|------------------|
|infra0|10.0.1.10|infra0.example.com|
|infra1|10.0.1.11|infra1.example.com|
|infra2|10.0.1.12|infra2.example.com|

用命令来启动 etcd 网关来使用这些静态终端:

```bash
$ etcd gateway start --endpoints=infra0.example.com,infra1.example.com,infra2.example.com
2016-08-16 11:21:18.867350 I | tcpproxy: ready to proxy client requests to [...]
```

或者，如果使用 DNS 做服务发现，考虑 DNS SRV entries:

```bash
$ dig +noall +answer SRV _etcd-client._tcp.example.com
_etcd-client._tcp.example.com. 300 IN SRV 0 0 2379 infra0.example.com.
_etcd-client._tcp.example.com. 300 IN SRV 0 0 2379 infra1.example.com.
_etcd-client._tcp.example.com. 300 IN SRV 0 0 2379 infra2.example.com.
```

```bash
$ dig +noall +answer infra0.example.com infra1.example.com infra2.example.com
infra0.example.com.  300  IN  A  10.0.1.10
infra1.example.com.  300  IN  A  10.0.1.11
infra2.example.com.  300  IN  A  10.0.1.12
```

用命令来启动 etcd 网关来从 DSN SRV entries 中获取终端：

```bash
$ etcd gateway --discovery-srv=example.com
2016-08-16 11:21:18.867350 I | tcpproxy: ready to proxy client requests to [...]
```

