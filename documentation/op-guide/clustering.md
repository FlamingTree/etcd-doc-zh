# 集群指南

> 注：内容翻译自 [Clustering Guide](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/clustering.md)

## 概述

启动 etcd 集群要求每个成员知道集群中的其他成员。在一些场景中，集群成员的 IP 地址可能无法提前知道。在这种情况下，etcd 集群可以在发现服务的帮助下启动。

一旦 etcd 集群启动并运行，通过 [运行时重配置](runtime-configuration.md) 来添加或者移除成员。为了更好的理解运行时重配置背后的设计，建议阅读 [运行时重配置的设计](runtime-reconf-design.md)。

这份指南将覆盖下列用于启动 etcd 集群的机制：

* [Static / 静态](#static)
* [etcd Discovery / etcd 发现](#etcd-discovery)
* [DNS Discovery / DNS 发现](#dns-discovery)

启动机制的每一种都将用于启动三台机器的 etcd 集群，详情如下：

|名字|地址|主机|
|------|---------|------------------|
|infra0|10.0.1.10|infra0.example.com|
|infra1|10.0.1.11|infra1.example.com|
|infra2|10.0.1.12|infra2.example.com|

## 静态

如果在启动前我们知道集群成员，他们的地址和集群的大小，我们可以使用通过设置 `initial-cluster` 标记来离线启动配置。每个机器将得到下列环境变量或者命令行：

```bash
ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380"
ETCD_INITIAL_CLUSTER_STATE=new
```

```bash
--initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
--initial-cluster-state new
```

注意： 在 `initial-cluster` 中指定的 URL 是 _advertised peer URLs_ ，例如，他们将匹配对应节点的 `initial-advertise-peer-urls` 的值。

如果使用同样的配置启动多个集群(或者创建并部署单个集群)用于测试目的，强烈推荐每个集群给予一个唯一的 `initial-cluster-token`。这样做之后，etcd 可以为集群生成唯一的集群 ID 和成员 ID，甚至他们有完全一样的配置。这可以将 etcd 从可能让集群孵化的跨集群交互中保护起来。

etcd 在 [`listen-client-urls`](configuration.md#--advertise-client-urls) 上接收客户端访问。etcd 成员将 [`advertise-client-urls`](configuration.md#--listen-client-urls) 指定的 URl 上通告给其他成员，代理和客户端。常见的错误是设置 `advertise-client-urls` 为 localhost 或者留空为默认值，如果远程客户端可以达到 etcd。

在每台机器上，使用这些标记启动 etcd：

```bash
$ etcd --name infra0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --listen-peer-urls http://10.0.1.10:2380 \
  --listen-client-urls http://10.0.1.10:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.10:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
```
```bash
$ etcd --name infra1 --initial-advertise-peer-urls http://10.0.1.11:2380 \
  --listen-peer-urls http://10.0.1.11:2380 \
  --listen-client-urls http://10.0.1.11:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.11:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
```
```bash
$ etcd --name infra2 --initial-advertise-peer-urls http://10.0.1.12:2380 \
  --listen-peer-urls http://10.0.1.12:2380 \
  --listen-client-urls http://10.0.1.12:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.12:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
```

以 `--initial-cluster` 开头的命令行参数将在 etcd 随后的运行中被忽略。可以在初始化启动进程之后随意的删除环境变量或者命令行标记。如果配置需要稍后修改(例如，添加成员到集群或者从集群中移除成员)，查看 [运行时配置](runtime-configuration.md) 指南。

### TLS

etcd 支持通过 TLS 协议的加密通讯。TLS 通道可以用于加密伙伴间的内部集群通讯，也可以用于加密客户端请求。这个章节提供例子来搭建使用伙伴和客户端 TLS 的集群。详细描述 etcd 的 TLS 支持的额外信息可以在 [加密指南](security.md)

#### 自签名证书

使用自签名证书证书(self-signed certificates)的集群同事加密请求并认证它的连接。要启动使用自签名证书的集群，每个集群成员应该有一个唯一的通过共享的集群CA证书来签名的键对(key pair) (`member.crt`, `member.key`)，用于伙伴连接和客户端连接。证书可以通过仿照 etcd [搭建TLS](https://github.com/coreos/etcd/blob/master/hack/tls-setup) 的例子来生成。

在每台机器上，etcd可以使用这些标记来启动：

```bash
$ etcd --name infra0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --listen-peer-urls https://10.0.1.10:2380 \
  --listen-client-urls https://10.0.1.10:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.1.10:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=https://10.0.1.10:2380,infra1=https://10.0.1.11:2380,infra2=https://10.0.1.12:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=/path/to/ca-client.crt \
  --cert-file=/path/to/infra0-client.crt --key-file=/path/to/infra0-client.key \
  --peer-client-cert-auth --peer-trusted-ca-file=ca-peer.crt \
  --peer-cert-file=/path/to/infra0-peer.crt --peer-key-file=/path/to/infra0-peer.key
```
```bash
$ etcd --name infra1 --initial-advertise-peer-urls https://10.0.1.11:2380 \
  --listen-peer-urls https://10.0.1.11:2380 \
  --listen-client-urls https://10.0.1.11:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.1.11:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=https://10.0.1.10:2380,infra1=https://10.0.1.11:2380,infra2=https://10.0.1.12:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=/path/to/ca-client.crt \
  --cert-file=/path/to/infra1-client.crt --key-file=/path/to/infra1-client.key \
  --peer-client-cert-auth --peer-trusted-ca-file=ca-peer.crt \
  --peer-cert-file=/path/to/infra1-peer.crt --peer-key-file=/path/to/infra1-peer.key
```
```bash
$ etcd --name infra2 --initial-advertise-peer-urls https://10.0.1.12:2380 \
  --listen-peer-urls https://10.0.1.12:2380 \
  --listen-client-urls https://10.0.1.12:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.1.12:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=https://10.0.1.10:2380,infra1=https://10.0.1.11:2380,infra2=https://10.0.1.12:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=/path/to/ca-client.crt \
  --cert-file=/path/to/infra2-client.crt --key-file=/path/to/infra2-client.key \
  --peer-client-cert-auth --peer-trusted-ca-file=ca-peer.crt \
  --peer-cert-file=/path/to/infra2-peer.crt --peer-key-file=/path/to/infra2-peer.key
```

#### 自动证书

如果集群需要加密通讯，但是不需要认证连接，etcd 可以配置为自动生成 key。在初始化时，每个集群基于它的通告IP(advertised IP) 地址和主机名创建它自己的 key 集合。

在每台机器上，etcd 使用这些标记启动：

```bash
$ etcd --name infra0 --initial-advertise-peer-urls https://10.0.1.10:2380 \
  --listen-peer-urls https://10.0.1.10:2380 \
  --listen-client-urls https://10.0.1.10:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.1.10:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=https://10.0.1.10:2380,infra1=https://10.0.1.11:2380,infra2=https://10.0.1.12:2380 \
  --initial-cluster-state new \
  --auto-tls \
  --peer-auto-tls
```
```bash
$ etcd --name infra1 --initial-advertise-peer-urls https://10.0.1.11:2380 \
  --listen-peer-urls https://10.0.1.11:2380 \
  --listen-client-urls https://10.0.1.11:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.1.11:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=https://10.0.1.10:2380,infra1=https://10.0.1.11:2380,infra2=https://10.0.1.12:2380 \
  --initial-cluster-state new \
  --auto-tls \
  --peer-auto-tls
```
```bash
$ etcd --name infra2 --initial-advertise-peer-urls https://10.0.1.12:2380 \
  --listen-peer-urls https://10.0.1.12:2380 \
  --listen-client-urls https://10.0.1.12:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.1.12:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=https://10.0.1.10:2380,infra1=https://10.0.1.11:2380,infra2=https://10.0.1.12:2380 \
  --initial-cluster-state new \
  --auto-tls \
  --peer-auto-tls
```

### 错误案例

#### 案例1

在下面的例子中，我们没有在列举的节点列表中包含我们新的主机地址。如果这是一个新的集群，节点 _必须_ 添加到初始化集群成员的列表中。

```bash
$ etcd --name infra1 --initial-advertise-peer-urls http://10.0.1.11:2380 \
  --listen-peer-urls https://10.0.1.11:2380 \
  --listen-client-urls http://10.0.1.11:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.11:2379 \
  --initial-cluster infra0=http://10.0.1.10:2380 \
  --initial-cluster-state new
etcd: infra1 not listed in the initial cluster config
exit 1
```

#### 案例2

在这个例子中，我们试图映射节点(infra0)在不同地址(127.0.0.1:2380)而不是它在集群列表(10.0.1.10:2380)中列举的地址。如果这个节点是监听多个地址，所有地址 _必须_ 在 "initial-cluster" 配置指令中反映出来。

```bash
$ etcd --name infra0 --initial-advertise-peer-urls http://127.0.0.1:2380 \
  --listen-peer-urls http://10.0.1.10:2380 \
  --listen-client-urls http://10.0.1.10:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.10:2379 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state=new
etcd: error setting up initial cluster: infra0 has different advertised URLs in the cluster and advertised peer URLs list
exit 1
```

#### 案例3

如果伙伴被用配置参数的不同集合配置并试图加入这个集群，etcd 将报告集群 ID 不匹配并退出。

```bash
$ etcd --name infra3 --initial-advertise-peer-urls http://10.0.1.13:2380 \
  --listen-peer-urls http://10.0.1.13:2380 \
  --listen-client-urls http://10.0.1.13:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.13:2379 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra3=http://10.0.1.13:2380 \
  --initial-cluster-state=new
etcd: conflicting cluster ID to the target cluster (c6ab534d07e8fcc4 != bc25ea2a74fb18b0). Exiting.
exit 1
```

## 发现

在一些案例中，集群伙伴的 IP 可能无法提前知道。当使用云提供商或者网络使用 DHCP 时比较常见。在这些情况下，相比指定静态配置，使用使用已经存在的 etcd 集群来启动一个新的。我们称这个过程为"发现"。

有两个方法可以用来做发现：

* etcd 发现服务
* DNS SRV 记录

### etcd 发现

为了更好的理解发现服务协议的设计，建议阅读发现服务项目 [文档](https://github.com/coreos/etcd/blob/master/Documentation/dev-internal/discovery_protocol.md)。

#### 发现 URL 的存活时间

发现 URL 标识唯一的 etcd 集群。对于新的集群，总是创建发现 URL 而不是重用发现 URL。

此外，发现URL应该仅仅用于集群的初始化启动。在集群已经运行之后修改集群成员，阅读 [运行时重配置](runtime-configuration.md) 指南。

#### 定制 etcd 发现服务

发现使用已有集群来启动自身。如果使用私有的 etcd 集群，可以创建像这样的 URL：


```bash
$ curl -X PUT https://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83/_config/size -d value=3
```

通过设置 URL 的 size，创建了带有期待集群大小为3的发现 URL。

用于这个场景的 URL 将是  `https://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83` 而 etcd 成员将使用 `https://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83` 目录来注册，当他们启动时。

** 每个成员必须有指定不同的名字标记。 `Hostname` 或者 `machine-id` 是个好选择。. 否则发现会因为重复名字而失败**

现在我们用这些用于每个成员的相关标记启动 etcd ：

```bash
$ etcd --name infra0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --listen-peer-urls http://10.0.1.10:2380 \
  --listen-client-urls http://10.0.1.10:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.10:2379 \
  --discovery https://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83
```
```bash
$ etcd --name infra1 --initial-advertise-peer-urls http://10.0.1.11:2380 \
  --listen-peer-urls http://10.0.1.11:2380 \
  --listen-client-urls http://10.0.1.11:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.11:2379 \
  --discovery https://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83
```
```bash
$ etcd --name infra2 --initial-advertise-peer-urls http://10.0.1.12:2380 \
  --listen-peer-urls http://10.0.1.12:2380 \
  --listen-client-urls http://10.0.1.12:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.12:2379 \
  --discovery https://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83
```

这将导致每个成员使用定制的 etcd 发现服务注册自身并开始集群，一旦所有的机器都已经注册。

#### 公共 etcd 发现服务

如果没有现成的集群可用，可以使用托管在 `discovery.etcd.io` 的公共发现服务。为了使用"new" endpoint来创建私有发现URL，使用命令：

```bash
$ curl https://discovery.etcd.io/new?size=3
https://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de
```

这将创建带有初始化预期大小为3个成员的集群。如果没有指定大小，将使用默认值3。

```bash
ETCD_DISCOVERY=https://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de
```

```bash
--discovery https://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de
```

** 每个成员必须有指定不同的名字标记。 `Hostname` 或者 `machine-id` 是个好选择。. 否则发现会因为重复名字而失败**

现在我们用这些用于每个成员的相关标记启动 etcd ：

```bash
$ etcd --name infra0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --listen-peer-urls http://10.0.1.10:2380 \
  --listen-client-urls http://10.0.1.10:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.10:2379 \
  --discovery https://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de
```
```bash
$ etcd --name infra1 --initial-advertise-peer-urls http://10.0.1.11:2380 \
  --listen-peer-urls http://10.0.1.11:2380 \
  --listen-client-urls http://10.0.1.11:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.11:2379 \
  --discovery https://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de
```
```bash
$ etcd --name infra2 --initial-advertise-peer-urls http://10.0.1.12:2380 \
  --listen-peer-urls http://10.0.1.12:2380 \
  --listen-client-urls http://10.0.1.12:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.12:2379 \
  --discovery https://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de
```

这将导致每个成员使用定制的etcd发现服务注册自身并开始集群，一旦所有的机器都已经注册。

使用环境变量 `ETCD_DISCOVERY_PROXY` 来让 etcd 使用 HTTP 代理来连接到发现服务。

#### 错误和警告案例

##### 发现服务错误

```bash
$ etcd --name infra0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --listen-peer-urls http://10.0.1.10:2380 \
  --listen-client-urls http://10.0.1.10:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.10:2379 \
  --discovery https://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de
etcd: error: the cluster doesn’t have a size configuration value in https://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de/_config
exit 1
```

##### 警告

这是一个无害的警告，表明发现 URL 将在这台机器上被忽略。

```bash
$ etcd --name infra0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --listen-peer-urls http://10.0.1.10:2380 \
  --listen-client-urls http://10.0.1.10:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.10:2379 \
  --discovery https://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de
etcdserver: discovery token ignored since a cluster has already been initialized. Valid log found at /var/lib/etcd
```

### DNS 发现

DNS [SRV records](http://www.ietf.org/rfc/rfc2052.txt) 可以作为发现机制使用。

`-discovery-srv` 标记可以用于设置 DNS domain name ，在这里可以找到发现 SRV 记录。

下列 DNS SRV 记录将以列出的顺序查找：

* _etcd-server-ssl._tcp.example.com
* _etcd-server._tcp.example.com

如果 `_etcd-server-ssl._tcp.example.com` 被发现则 etcd 将在 TLS 上启动尝试启动进程。

为了帮助客户端发现 etcd 集群，下列 DNS SRV 记录将以列出的顺序查找：

* _etcd-client._tcp.example.com
* _etcd-client-ssl._tcp.example.com

如果发现 `_etcd-client-ssl._tcp.example.com` , 客户端将在 SSL/TLS 上尝试和etcd集群通讯。

如果 etcd 不带定义的证书 authority 使用 TLS，发现域名(如, example.com) 必须匹配 SRV record domain (例如, infra1.example.com). 这是为了减轻仿制 SRV 记录来指向不同域名的损害; 域名可能有 PKI 之下的有效证书，但是被未知的第三方控制。

#### 创建 DNS SRV 记录

```bash
$ dig +noall +answer SRV _etcd-server._tcp.example.com
_etcd-server._tcp.example.com. 300 IN  SRV  0 0 2380 infra0.example.com.
_etcd-server._tcp.example.com. 300 IN  SRV  0 0 2380 infra1.example.com.
_etcd-server._tcp.example.com. 300 IN  SRV  0 0 2380 infra2.example.com.
```

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

#### 使用 DNS 启动 etcd 集群

etcd 集群成员可以监听域名或者IP地址，启动进程将解析 DNS A 记录。


在 `--initial-advertise-peer-urls` 中解析出来的地址 *必须匹配* 在 SRV 目录中解析出来的地址中的一个。 etcd 成员读取被解析的地址来发现它是否属于 SRV 记录定义的集群。

```bash
$ etcd --name infra0 \
--discovery-srv example.com \
--initial-advertise-peer-urls http://infra0.example.com:2380 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster-state new \
--advertise-client-urls http://infra0.example.com:2379 \
--listen-client-urls http://infra0.example.com:2379 \
--listen-peer-urls http://infra0.example.com:2380
```

```bash
$ etcd --name infra1 \
--discovery-srv example.com \
--initial-advertise-peer-urls http://infra1.example.com:2380 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster-state new \
--advertise-client-urls http://infra1.example.com:2379 \
--listen-client-urls http://infra1.example.com:2379 \
--listen-peer-urls http://infra1.example.com:2380
```

```bash
$ etcd --name infra2 \
--discovery-srv example.com \
--initial-advertise-peer-urls http://infra2.example.com:2380 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster-state new \
--advertise-client-urls http://infra2.example.com:2379 \
--listen-client-urls http://infra2.example.com:2379 \
--listen-peer-urls http://infra2.example.com:2380
```

集群也可以使用 IP 地址而不是域名来启动：

```bash
$ etcd --name infra0 \
--discovery-srv example.com \
--initial-advertise-peer-urls http://10.0.1.10:2380 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster-state new \
--advertise-client-urls http://10.0.1.10:2379 \
--listen-client-urls http://10.0.1.10:2379 \
--listen-peer-urls http://10.0.1.10:2380
```

```bash
$ etcd --name infra1 \
--discovery-srv example.com \
--initial-advertise-peer-urls http://10.0.1.11:2380 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster-state new \
--advertise-client-urls http://10.0.1.11:2379 \
--listen-client-urls http://10.0.1.11:2379 \
--listen-peer-urls http://10.0.1.11:2380
```

```bash
$ etcd --name infra2 \
--discovery-srv example.com \
--initial-advertise-peer-urls http://10.0.1.12:2380 \
--initial-cluster-token etcd-cluster-1 \
--initial-cluster-state new \
--advertise-client-urls http://10.0.1.12:2379 \
--listen-client-urls http://10.0.1.12:2379 \
--listen-peer-urls http://10.0.1.12:2380
```

### 网关

etcd 网关是一个简单的 TCP 代码，转发网络数据到 etcd 集群。请阅读 [网关指南]()来获取更多信息。

### 代理

当 `--proxy` 标记被设置时，etcd 运行于 [代理模式](https://github.com/coreos/etcd/blob/release-2.3/Documentation/proxy.md)。这个代理模式仅仅支持 etcd v2 API; 没有支持 v3 API 的计划。取而代之的是，对于 v3 API 支持，在 etcd 3.0 发布之后将会有一个新的有增强特性的代理。

为了搭建带有v2 API的代理的 etcd 集群，请阅读 [clustering doc in etcd 2.3 release](https://github.com/coreos/etcd/blob/release-2.3/Documentation/clustering.md).
