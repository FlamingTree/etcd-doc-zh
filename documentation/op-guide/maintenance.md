# 维护

> 注： 内容翻译自 [Maintenance](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/maintenance.md)

## 概述

etcd 集群需要定期维护来保持可靠。基于 etcd 应用的需要，这个维护通常可以自动执行，不需要停机或者显著的降低性能。

所有 etcd 的维护是指管理被 etcd 键空间消耗的存储资源。通过存储空间的配额来控制键空间大小;如果 etcd 成员运行空间不足，将触发集群级警告，这将使得系统进入有限操作的维护模式。为了避免没有空间来写入键空间， etcd 键空间历史必须被压缩。存储空间自身可能通过碎片整理 etcd 成员来回收。最后，etcd 成员状态的定期快照备份使得恢复任何非故意的逻辑数据丢失或者操作错误导致的损坏变成可能。

## 历史压缩

因为 etcd 保持它的键空间的确切历史，这个历史应该定期压缩来避免性能下降和最终的存储空间枯竭。压缩键空间历史删除所有关于被废弃的在给定键空间修订版本之前的键的信息。这些key使用的空间随机变得可用来继续写入键空间。

键空间可以使用 `etcd` 的时间窗口历史保持策略自动压缩，或者使用 `etcdctl` 手工压缩。 `etcdctl` 方法在压缩过程上提供细粒度的控制，反之自动压缩适合仅仅需要一定时间长度的键历史的应用。

`etcd` 可以使用带有小时时间单位的 `--auto-compaction` 选项来设置为自动压缩键空间:

```bash
# 保持一个小时的历史
$ etcd --auto-compaction-retention=1
```

`etcdctl` 如下发起压缩工作:

```bash
# 压缩到修订版本3
$ etcdctl compact 3
```

在压缩修订版本之前的修订版本变得无法访问：

```bash
$ etcdctl get --rev=2 somekey
Error:  rpc error: code = 11 desc = etcdserver: mvcc: required revision has been compacted
```

## 反碎片化

在压缩键空间之后，后端数据库可能出现内部。内部碎片是指可以被后端使用但是依然消耗存储空间的空间。反碎片化过程释放这个存储空间到文件系统。反碎片化在每个成员上发起，因此集群范围的延迟尖峰(latency spike)可能可以避免。

通过留下间隔在后端数据库，压缩旧有修订版本会内部碎片化 `etcd` 。碎片化的空间可以被 `etcd` 使用，但是对于主机文件系统不可用。

为了反碎片化 etcd 成员， 使用 `etcdctl defrag` 命令:

```bash
$ etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]
```

## 空间配额

在 `etcd` 中空间配额确保集群以可靠方式运作。没有空间配额， `etcd` 可能会收到低性能的困扰，如果键空间增长的过度的巨大，或者可能简单的超过存储空间，导致不可预测的集群行为。如果键空间的任何成员的后端数据库超过了空间配额， `etcd` 发起集群范围的警告，让集群进入维护模式，仅接收键的读取和删除。在键空间释放足够的空间之后，警告可以被解除，而集群将恢复正常运作。

默认，`etcd` 设置适合大多数应用的保守的空间配额，但是它可以在命令行中设置，单位为字节：

```bash
# 设置非常小的 16MB 配额
$ etcd --quota-backend-bytes=16777216
```

空间配额可以用循环触发：

```bash
# 消耗空间
$ while [ 1 ]; do dd if=/dev/urandom bs=1024 count=1024  | etcdctl put key  || break; done
...
Error:  rpc error: code = 8 desc = etcdserver: mvcc: database space exceeded
# 确认配额空间被超过
$ etcdctl --write-out=table endpoint status
+----------------+------------------+-----------+---------+-----------+-----------+------------+
|    ENDPOINT    |        ID        |  VERSION  | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+----------------+------------------+-----------+---------+-----------+-----------+------------+
| 127.0.0.1:2379 | bf9071f4639c75cc | 2.3.0+git | 18 MB   | true      |         2 |       3332 |
+----------------+------------------+-----------+---------+-----------+-----------+------------+
# 确认警告已发起
$ etcdctl alarm list
memberID:13803658152347727308 alarm:NOSPACE
```

删除多读的键空间将把集群带回配额限制，因此警告能被接触:

```bash
# 获取当前修订版本
$ etcdctl --endpoints=:2379 endpoint status
[{"Endpoint":"127.0.0.1:2379","Status":{"header":{"cluster_id":8925027824743593106,"member_id":13803658152347727308,"revision":1516,"raft_term":2},"version":"2.3.0+git","dbSize":17973248,"leader":13803658152347727308,"raftIndex":6359,"raftTerm":2}}]
# 压缩所有旧有修订版本
$ etdctl compact 1516
compacted revision 1516
# 反碎片化过度空间
$ etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]
# 解除警告
$ etcdctl alarm disarm
memberID:13803658152347727308 alarm:NOSPACE 
# 测试put被再度容许
$ etdctl put newkey 123
OK
```

## 快照备份

在正规基础上执行 `etcd` 集群快照可以作为 etc 键空间的持久备份。通过获取 etcd 成员的候选数据库的定期快照，`etcd` 集群可以被恢复到某个有已知良好状态的时间点。

通过 `etcdctl` 获取快照：

```bash
$ etcdctl snapshot save backup.db
$ etcdctl --write-out=table snapshot status backup.db
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| fe01cf57 |       10 |          7 | 2.1 MB     |
+----------+----------+------------+------------+

```