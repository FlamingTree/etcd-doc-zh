# 维护

> 注： 内容翻译自 [Maintenance](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/maintenance.md)

## 概述

etcd 集群需要定期维护以保持可靠性。基于 etcd 应用的需要，这个维护通常可以自动执行，不需要停机或者显著的降低性能。

所有 etcd 的维护是指管理 etcd 键空间消耗的存储资源。通过存储空间的配额来控制键空间大小；如果 etcd 成员运行空间不足，将触发集群级警告，这将使得系统进入操作受限的维护模式。为了避免耗尽可写键空间，需要压缩 etcd 键空间历史数据。存储空间可能通过碎片整理被回收。最后，对 etcd 成员状态进行定期快照备份，从而可以对操作失误造成的数据丢失和损坏进行恢复。

## 历史压缩

因为 etcd 存储键空间的所有历史数据，这个历史应该定期压缩来避免性能下降和最终的存储空间枯竭。压缩键空间历史将删除给定键空间修订版本之前的历史数据。被删除的键空间可以被新的数据写入。

键空间可以使用 `etcd` 的时间窗口历史保持策略自动压缩，或者使用 `etcdctl` 手动压缩。`etcdctl` 方法在压缩时提供细粒度的控制，反之自动压缩适用于仅需要一定时间长度的键历史的应用。

`etcd` 可以使用带有小时时间单位的 `--auto-compaction` 选项来设置为自动压缩:

```bash
# 保持一个小时的历史
$ etcd --auto-compaction-retention=1
```

`etcdctl` 启动压缩:

```bash
# 压缩到修订版本3
$ etcdctl compact 3
```

在压缩修订版本之前的修订版本变得无法访问：

```bash
$ etcdctl get --rev=2 somekey
Error:  rpc error: code = 11 desc = etcdserver: mvcc: required revision has been compacted
```

## 碎片整理（Defragmentation）

在压缩键空间之后，后端数据库内部可能出现碎片。内部碎片是指可以被后端使用但是依然消耗存储空间的空间。碎片整理释放这个存储空间到文件系统。碎片整理在每个成员上进行，从而避免集群上的延迟增加(latency spike)。

压缩旧版本数据出现的数据空洞就是碎片。碎片化的空间可以被 `etcd` 使用，但是对于主机文件系统不可用。

为了对 etcd 成员进行碎片整理，使用 `etcdctl defrag` 命令:

```bash
$ etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]
```

## 空间配额

`etcd` 中的空间配额确保集群以可靠方式运作。没有空间配额， `etcd` 可能会造成性能低下，如果键空间过度增长，或者耗尽存储空间，导致不可预测的集群行为。如果任何成员的键空间数据库超过了配额，`etcd` 发起集群范围的警告，让集群进入维护模式，仅接收读和删除操作。在释放的键空间足够大或碎片整理之后，警告可以被解除，而集群将恢复正常运作。

默认，`etcd` 设置适合大多数应用的保守的空间配额，但是它可以在命令行中设置，单位为字节：

```bash
# 设置非常小的 16MB 配额
$ etcd --quota-backend-bytes=$((16*1024*1024))
```

循环填充数据出发配额告警：

```bash
# 消耗空间
$ while [ 1 ]; do dd if=/dev/urandom bs=1024 count=1024  | ETCDCTL_API=3 etcdctl put key  || break; done
...
Error:  rpc error: code = 8 desc = etcdserver: mvcc: database space exceeded
# 确认超过空间配额
$ ETCDCTL_API=3 etcdctl --write-out=table endpoint status
+----------------+------------------+-----------+---------+-----------+-----------+------------+
|    ENDPOINT    |        ID        |  VERSION  | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+----------------+------------------+-----------+---------+-----------+-----------+------------+
| 127.0.0.1:2379 | bf9071f4639c75cc | 2.3.0+git | 18 MB   | true      |         2 |       3332 |
+----------------+------------------+-----------+---------+-----------+-----------+------------+
# 确认警告已发起
$ ETCDCTL_API=3 etcdctl alarm list
memberID:13803658152347727308 alarm:NOSPACE 
```

删除超过配额的键空间并整理碎片，把存储空间控制在限额内:

```bash
# 获取当前修订版本
$ rev=$(ETCDCTL_API=3 etcdctl --endpoints=:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9]*')
# 压缩旧版本
$ ETCDCTL_API=3 etcdctl compact $rev
compacted revision 1516
# 碎片整理
$ ETCDCTL_API=3 etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]
# 解除告警
$ ETCDCTL_API=3 etcdctl alarm disarm
memberID:13803658152347727308 alarm:NOSPACE 
# 测试允许 put 操作 
$ ETCDCTL_API=3 etcdctl put newkey 123
OK
```

## 快照备份

定期快照对 etcd 起到持久化存储的作用。通过定期对 etcd 进行快照，`etcd` 集群可以被恢复到某个已知良好状态的时间点。

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

