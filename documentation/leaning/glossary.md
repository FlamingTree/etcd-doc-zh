# 术语

> 注： 内容翻译自 [Glossary](https://github.com/coreos/etcd/blob/master/Documentation/learning/glossary.md)

此文档定义etcd文档，命令行和源代码中使用的多个术语。

## Alarm / 告警

当需要人为干预从而保持集群的稳定性时，etcd 服务器会发出告警。

## Authentication / 认证

认证对 etcd 资源的访问权限进行管理。

## Client / 客户端

客户端用于连接 etcd 服务器，从而发去请求（如获取键值，写入数据，watch）。

> 注： 对于V3,应该包括gRPC API。

## Cluster / 集群

Cluster/集群由多个member/成员组成。

每个成员节点遵循 raft 一致性协议来复制日志。集群从成员中接收提案，提交并应用到本地存储。

## Node / 节点

Node/节点是raft状态机的一个实例。

它有唯一标识，并内部记录其他节点的发展，如果它是leader。

## Member / 成员

Member/成员是etcd的一个实例。它承载一个node/节点，并为client/客户端提供服务。

## Compaction / 压缩

压缩丢弃 etcd 历史数据和指定版本之前的数据更新。它用来回收 etcd 后端数据库的存储空间。

## Election / 选举

按照 Raft 一致性协议，etcd 集群在成员之间进行选举从而远处 leader。

## Endpoint / 终端

指向 etcd 服务或资源的 URL。

## Key / 键

用户定义的用来存储和检索数据的 ID。

## Key range / 区间键

包含单独的键，位于字典排序区间 a < x <= b 或者 大于指定 key 的 x 值 的键的集合。

## Keyspace / 键空间

etcd 集群中的所有键。

## Lease / 租约

短期的、可更新的约定，当它过过期时会删除所关联的键。

## Member / 成员

作为 etcd 集群服务一部分的 etcd 服务器。

## Peer / 同伴

Peer/同伴是同一个集群中的其他成员。

## Proposal / 提议

提议是一个需要通过 Raft 协议完成的请求(例如写请求，配置修改请求)。

## Quorum / 法定数量

当改变 etcd 状态时，为了达到一致性所需的活跃成员数量。etcd 需要多数成员构成法定数量。

## Revision / 修订版本

集群范围内的计数器，每次键空间修改都会增加。

## Role / 角色

授予某些用户的对于某些键的访问权限。

## Snapshot / 快照

etcd 集群状态在某个时间点的备份。

## Store / 存储

etcd 键空间的物理存储。

## Transaction / 事务

一组操作的原子执行。在一个事务中所有被修改的间公用一个修改版本。

## Key Version / 键版本

键创建之后被写入的次数，从1开始。不存在或被删除的键版本是0。

## Watcher / 观察者

对某些键进行观察的客户端。


