# 灾难恢复

> 注： 内容翻译自 [Disaster recovery](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/recovery.md)

etcd 能承受机器故障。etcd 集群自动从临时失败(例如，机器重启)中恢复，而且对于一个有 N 个成员的集群可容许 *(N-1)/2* 的机器故障。当一个成员永久故障时，不管是因为硬件故障或者磁盘损坏，它和集群失去联系。如果集群永久性丢失超过 *(N-1)/2* 的成员，会造成灾难性故障，无法达到法定人数(quorum)。一旦无法达到法定人数，集群无法达到一致从而更新数据。

为了从灾难故障中恢复，etcd v3 提供快照和修复工具来重建集群而不丢失 v3 版本的数据。要恢复 v2 版本的数据，参考[v2 管理指南](https://github.com/coreos/etcd/blob/master/Documentation/v2/admin_guide.md#disaster-recovery).

### 快照键空间

恢复集群首先需要来自 etcd 成员键空间的快照。快速可以用 `etcdctl snapshot save` 命令从活的成员获取，或者是从 etcd 数据目录复制 `member/snap/db` 文件。例如，下列命令把 `$ENDPOINT` 上键空间的快照文件保存`snapshot.db`:

```bash
$ etcdctl --endpoints $ENDPOINT snapshot save snapshot.db
```

### 恢复集群

为了恢复集群，仅需要一个快照文件 "db"。通过 `etcdctl snapshot restore` 命令恢复的集群创建新的 etcd 数据目录；所有成员应该使用相同的快照进行恢复。恢复重写某些快照元数据(特别是，成员ID和集群ID)；成员丢失它之前的标识。这个元数据重写防止新的成员不经意间加入已有的集群。因此为了从快照启动集群，恢复必须启动一个新的逻辑集群。

在恢复时快照完整性的检验是可选的。如果快照是通过 `etcdctl snapshot save` 得到的，它将有一个完整性hash，在 `etcdctl snapshot restore` 时会对它进行验证。如果快照是从数据目录复制而来，它不存在完整性hash，因此它只能通过使用 `--skip-hash-check` 来恢复。

恢复初始化新集群的新成员，使用 `etcd` 的集群标记设置新的配置，但是保存 etcd 键空间的内容。继续上面的例子，下面为一个3成员的集群创建新的 etcd 数据目录(`m1.etcd`, `m2.etcd`, `m3.etcd`):

```bash
$ etcdctl snapshot restore snapshot.db \
  --name m1 \
  --initial-cluster m1=http://host1:2380,m2=http://host2:2380,m3=http://host3:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls http://host1:2380
$ etcdctl snapshot restore snapshot.db \
  --name m2 \
  --initial-cluster m1=http://host1:2380,m2=http://host2:2380,m3=http://host3:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls http://host2:2380
$ etcdctl snapshot restore snapshot.db \
  --name m3 \
  --initial-cluster m1=http://host1:2380,m2=http://host2:2380,m3=http://host3:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls http://host3:2380
```

下一步, 用新的数据目录启动 `etcd` :

```bash
$ etcd \
  --name m1 \
  --listen-client-urls http://host1:2379 \
  --advertise-client-urls http://host1:2379 \
  --listen-peer-urls http://host1:2380 &
$ etcd \
  --name m2 \
  --listen-client-urls http://host2:2379 \
  --advertise-client-urls http://host2:2379 \
  --listen-peer-urls http://host2:2380 &
$ etcd \
  --name m3 \
  --listen-client-urls http://host3:2379 \
  --advertise-client-urls http://host3:2379 \
  --listen-peer-urls http://host3:2380 &
```

现在恢复的集群可以使用，并且使用快照的数据进行服务。

