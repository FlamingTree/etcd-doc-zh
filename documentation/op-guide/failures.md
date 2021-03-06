# 理解失败

> 注： 内容翻译自 [Understand failures](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/failures.md)

在部署规模较大的机器中失败是很常见的。当硬件或者软件故障时单台机器失败。当电力故障或者网络问题时多台机器同时失败。多种失败也可能一起发生；几乎不可能列举出所有可能的失败场景。

在这节中，我们对失败进行分类并讨论 etcd 是如何设计来容忍这些失败的。对于大多说用户，可以把遇到的失败归结到其中的一种。为了应对罕见或者 [不可恢复的失败][unrecoverable], 需要 [备份][backup] etcd 集群。

## 少数 follower 失败

当少于一半的 follower 失败时，etcd 集群依然可以不中断的接收请求。例如，两个 follower 失败将不会影响一个五个成员的 etcd 集群运作。但是，客户端到失败成员的连接将会断开。对于请求客户端类库应该通过自动重新连接到其他成员，对用户隐藏这些中断。运维人员应该预期其他成员上的系统负载会因为重连而提高。

## Leader 失败

当 leader 失败时，etcd 集群自动选举一个新的 leader。选举不会在 leader 失败之后立即发生。在一个选举超时的时间后将会开始选举新的 leader，因为失败检测模型是基于超时的。

在 leader 选举的期间，集群不能处理任何写操作。在新的 leader 被选举出来之前，所有写请求将会排队。

已经发送给旧有 leader 但是还没有提交的写请求可能会丢失。新的 leader 有权力重写来自旧 leader尚未被提交的数据。从用户的角度，某些写请求可能在新 leader 被选出后超时。无论如何，已提交的请求从来不会丢失。

新的 leader 自动延长所有的租约(lease)。这个机制保证租约将不会在授予的 TTL 之前过期，即使它是被旧有的 leader 授予。

## 多数失败

当集群的大多数成员失败时，etcd 集群失败并无法接收更多写请求。

一旦多数成员重新可用后，etcd 集群就会从多数失败中恢复。如果多数成员无法连通，那么运维必须启动[disaster recovery/灾难恢复][unrecoverable] 来恢复集群。

一旦多数成员开始工作， etcd 集群会自动选举新的 leader 并进入到健康状态。新的 leader 自动延长所有租约的超时时间。这个机制确保没有租约因为服务器端不可访问而过期。

## 网络分区

网络分区类似少数 follower 失败或者 leader 失败。网络分区将 etcd 集群分成两个部分; 一个有多数成员而另外一个有少数成员。多数这边变成可用集群而少数这边不可用。在 etcd 中没有 "脑裂"。

如果 leader 在多数这边，那么从多数这边的角度看失败是一个少数 follower 失败。如果 leader 在少数这边，那么它是 leader 失败。在少数这边的 leader 下台(step down)然后多数那边选举新的 leader。

一旦网络分区清除，少数这边自动承认来自多数这边的 leader 并恢复状态。

## 启动期间失败

集群启动时仅仅当所有要求的成员都成功启动才视为成功。如果在启动期间发生任何失败，在所有成员上删除数据目录并用新的集群记号(cluster-token)或者新的 discovery 记号(discovery token)重新启动集群。

当然，可以像恢复运行中的集群那样恢复失败的启动集群。但是，大多数情况下恢复一个集群比重新启动一个集群花费更多的资源和时间，因为不需要恢复任何数据。

[backup]: maintenance.md#快照备份
[unrecoverable]: recovery.md#灾难恢复

