# 运行时重配置

> 注：内容翻译自 [Runtime Reconfiguration](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/runtime-configuration.md)

etcd 自带对渐进的运行时重配置的支持，这容许用户在运行时更新集群成员。

重配置仅能在集群成员的大多数正常工作时才可以进行。**强烈推荐** 在生产环境中集群大小总是大于2。从两节点集群中移除一个节点是不安全的。在移除过程中如果有失败，集群可能无法正常运行而需要[从多说节点失败中重启][majority failure].

为了更好的理解运行时重配置后面的设计，建议阅读 [运行时重配置文档][runtime-reconf].

## 重配置使用案例

让我们过一下一些重配置集群的常见理由。大多数仅仅涉及添加或者移除成员，这些将在后面的 [集群重配置操作][cluster-reconf] 中解释。

### 升级多台机器

如果多个集群成员因为计划的维护(硬件升级，网络停工)而需要移动，推荐逐个更改成员。

移除 leader 是安全的，但是在选举过程发生的期间有短暂的停机时间。如果集群保存超过50MB，推荐 [迁移成员的数据目录][member migration]。

### 修改集群大小

增加集群大小可以改善 [失败容忍度][fault tolerance table] 并提供更好的读取性能。因为客户端可以从任意成员读取，增加成员的数量可以提高整体的读取吞吐量。

减少集群大小可以改善集群的写入性能，但是会降低失败后的恢复能力。写入的数据需要复制到集群中的大多数节点后才被认为正确提交。集群规模的减小导致复制数量的减少，从而每次写入可以更快提交。

### 替换失败的机器

如果机器因为硬件故障，数据目录损坏，或者一些其他致命情况而失败，它应该尽快被替代。已经失败但是还没有移除的机器对Raft协议法定人数有不利影响并减低对额外失败的容忍性。

为了替换机器，遵循从集群中 [移除成员][remove member] 的建议, 然后再 [添加新成员][add member]。 如果集群数据量超过50MB, 推荐 [迁移失败成员的数据目录][member migration]，如果它还可以访问。

### 从多数节点失败中重启集群

如果集群的多数已经丢失或者所有的节点已经修改了IP地址，则需要手工安全恢复。
恢复过程中的基本步骤包括 [使用旧有数据创建新的集群][disaster recovery], 强制此几点成为 leader，并最终使用运行时配置来逐个 [添加新的成员][add member] 到这个新的集群。

## 集群重配置操作

现在我们心里有使用案例了，让我们展示每个案例中涉及到的操作。

在任何变动前，etcd 成员的大多说(quorum) 必须可用。本质上和 etcd 集群可写的要求相同。

所有集对群的改动逐次完成:

* 要更新单个成员peerURLs，做一个更新操作
* 要替换单个成员，做一个添加然后一个删除操作
* 要将节点数量从3增加到5，做两次添加操作
* 要将节点数量从5减少到3，做两次删除操作

所有这些案例将使用etcd自带的 `etcdctl` 命令行工具。
如果不用 `etcdctl` 修改成员，可以使用 [v2 HTTP members API][member-api] 或者 [v3 gRPC members API][member-api-grpc].

### 更新成员

#### 更新 advertise client URLs

为了更新节点的 advertise client URLs，使用新的 client URL 标记(`--advertise-client-urls`)或者环境变量(`ETCD_ADVERTISE_CLIENT_URLS`)重启节点即可。重新后的成员将自行发布更新后的URL。错误更新的client URL 将不会影响 etcd 集群的健康。

#### 更新 advertise peer URLs

要更新节点的 advertise peer URLs, 首先通过 member 命令更新它然后再重启节点。需要额外的操作是因为更新 peer URL 修改了集群级别的配置并会影响 etcd 集群的健康。

要更新 peer URL，首先，我们需要找到目标成员的ID。使用 `etcdctl` 列出所有成员：

```bash
$ etcdctl member list
6e3bd23ae5f1eae0: name=node2 peerURLs=http://localhost:23802 clientURLs=http://127.0.0.1:23792
924e2e83e93f2560: name=node3 peerURLs=http://localhost:23803 clientURLs=http://127.0.0.1:23793
a8266ecf031671f3: name=node1 peerURLs=http://localhost:23801 clientURLs=http://127.0.0.1:23791
```

在这个例子中，让我们 `更新` a8266ecf031671f3 成员ID并修改它的 peerURLs 值为 http://10.0.1.10:2380。

```bash
$ etcdctl member update a8266ecf031671f3 http://10.0.1.10:2380
Updated member with ID a8266ecf031671f3 in cluster
```

### 删除成员

假设我们要删除的成员ID是 a8266ecf031671f3。
我们随后用 `remove` 命令来进行删除:

```bash
$ etcdctl member remove a8266ecf031671f3
Removed member a8266ecf031671f3 from cluster
```

此时目标成员将自己停止并在日志中打印出移除信息：

```
etcd: this member has been permanently removed from the cluster. Exiting.
```

移除 leader是安全的，当然在新 leader 被选举时期间集群将不生效(inactive)。这个持续时间通常是选举超时时间加投票花费的时间。

### 添加新成员

添加成员的过程有两个步骤：

* 通过 [HTTP members API][member-api] 添加新成员到集群, [gRPC members API][member-api-grpc], 或者 `etcdctl member add` 命令.
* 使用新配置启动新成员，包括更新后的成员列表(以后成员加新成员)

使用 `etcdctl` 指定 [name][conf-name] 和 [advertised peer URLs][conf-adv-peer] 来添加新的成员到集群:

```bash
$ etcdctl member add infra3 http://10.0.1.13:2380
added member 9bf1b35fc7761a23 to cluster

ETCD_NAME="infra3"
ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra3=http://10.0.1.13:2380"
ETCD_INITIAL_CLUSTER_STATE=existing
```

`etcdctl` 已经给出关于新成员的集群信息，并打印出成功启动它所需要的环境变量。现在用相应的参数启动新的节点:

```bash
$ export ETCD_NAME="infra3"
$ export ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra3=http://10.0.1.13:2380"
$ export ETCD_INITIAL_CLUSTER_STATE=existing
$ etcd --listen-client-urls http://10.0.1.13:2379 --advertise-client-urls http://10.0.1.13:2379 --listen-peer-urls http://10.0.1.13:2380 --initial-advertise-peer-urls http://10.0.1.13:2380 --data-dir %data_dir%
```

新成员将作为集群的一部分运行并立即开始同步数据。

如果添加多个成员，最佳实践是添加一个成员并验证成功后，再添加其他成员。
如果添加新成员到单节点的集群，在新成员启动前集群无法继续工作，因为它至少需要两个成员才能达成一致性。这个行为仅仅发生在 `etcdctl member add` 通知集群后和新旧成员建立连接之前。

#### 添加成员时的错误案例

在下面的案例中，我们没有在节点列表中包含新的host。新节点必须添加到 `initial-cluster` 列表中。

```bash
$ etcd --name infra3 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state existing
etcdserver: assign ids error: the member count is unequal
exit 1
``````

在这个案例中，给出的地址(10.0.1.14:2380)与加入集群指定的地址(10.0.1.13:2380)不同

```bash
$ etcd --name infra4 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380,infra4=http://10.0.1.14:2380 \
  --initial-cluster-state existing
etcdserver: assign ids error: unmatched member while checking PeerURLs
exit 1
```

使用已被移除成员的数据目录来启动 etcd 时，当它连接到集群中的任何活动成员后，etcd将立即退出：

```sh
$ etcd
etcd: this member has been permanently removed from the cluster. Exiting.
exit 1
```

### 严格重配置检查模式 (`-strict-reconfig-check`)

如上所述，添加新成员的最佳实践是添加一个成员并验证成功后，再添加其他成员。这个单步的方式非常重要，因为如果最新添加的成员没有正确配置(例如 peer URL 不正确)，集群会丢失法定人数（quorum）。发生法定人数丢失是因为最新加入的成员被法定人数计数，即使其他已经存在的成员无法访问它。同样法定人数丢失可能发生在有连接问题或者操作问题时。

为了避免这个问题，etcd 提供选项 `-strict-reconfig-check`。如果这个选项被传递给 etcd，如果已启动成员的数量少于被重配置后的集群法定人数，etcd 将拒绝重配置请求。

此选项默认打开。

[add member]: #添加新成员
[cluster-reconf]: #集群重配置操作
[conf-adv-peer]: configuration.md#-initial-advertise-peer-urls
[conf-name]: configuration.md#-name
[disaster recovery]: recovery.md
[fault tolerance table]: https://github.com/coreos/etcd/blob/master/Documentation/v2/admin_guide.md#fault-tolerance-table
[majority failure]: #从多数节点失败中重启集群
[member-api]: https://github.com/coreos/etcd/blob/master/Documentation/v2/members_api.md
[member-api-grpc]: https://github.com/coreos/etcd/blob/master/Documentation/dev-guide/api_reference_v3.md#service-cluster-etcdserveretcdserverpbrpcproto
[member migration]: https://github.com/coreos/etcd/blob/master/Documentation/v2/admin_guide.md#member-migration
[remove member]: #删除成员
[runtime-reconf]: runtime-reconf-design.md

