# 术语

> 注： 内容翻译自 [Glossary](https://github.com/coreos/etcd/blob/master/Documentation/learning/glossary.md)

这份文档定义etcd文档，命令行和源代码中使用的多个术语。

## Node / 节点

Node/节点是raft状态机的一个实例。

它有唯一标识，并内部记录其他节点的发展，如果它是leader。

## Member / 成员

Member/成员是etcd的一个实例。它承载一个node/节点，并为client/客户端提供服务。

## Cluster / 集群

Cluster/集群由多个member/成员组成。

每个成员的节点遵循 raft 一致性协议来复制日志。集群从成员中接收提案，提交他们并应用到本地存储。

## Peer / 同伴

Peer/同伴是同一个集群中的其他成员。

## Proposal / 提议

提议是一个需要完成raft协议的请求(例如写请求，配置修改请求)。

## Client / 客户端

Client/客户端是集群HTTP API的调用者。

> 注： 对于V3,应该包括gRPC API。

## Machine / 机器 (弃用)

在 2.0 版本之前，在etcd中的成员备选。
