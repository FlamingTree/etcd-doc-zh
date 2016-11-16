# 和 etcd 交互

> 注： 内容翻译自 [Interacting with etcd](https://github.com/coreos/etcd/blob/master/Documentation/dev-guide/interacting_v3.md)

用户通常通过设置或者获取 key 的值来和 etcd 交互。这一节描述如何使用 etcdctl 来操作， etcdctl 是一个和 etcd 服务器交互的命令行工具。这里描述的概念也适用于 gRPC API 或者客户端类库 API。

默认，为了向后兼容 etcdctl 使用 v2 API 来和 etcd 服务器通讯。为了让 etcdctl 使用 v3 API 来和etcd通讯，API 版本必须通过环境变量 `ETCDCTL_API` 设置为版本3。

``` bash
export ETCDCTL_API=3
```

## 写入key

应用通过写入 key 来储存 key 到 etcd 中。每个存储的 key 被通过 Raft 协议复制到所有 etcd 集群成员来达到一致性和可靠性。

这是设置 key `foo` 的值为 `bar` 的命令:


``` bash
$ etcdctl put foo bar
OK
```

## 读取 key

应用可以从 etcd 集群中读取 key 的值。查询可以读取单个 key，或者某个范围的 key。

假设 etcd 集群存储有下面的 key：

```bash
foo = bar
foo1 = bar1
foo3 = bar3
```

这是读取 key `for` 的值的命令：

```bash
$ etcdctl get foo
foo
bar
```

这是覆盖从 `foo` to `foo9` 的 key 的命令：

```bash
$ etcdctl get foo foo9
foo
bar
foo1
bar1
foo3
bar3
```

> 注： 测试中发现，这个命令的有效区间是这样：`[foo foo9)`， 即包含第一个参数，但是不包含第二个参数。因此如果第二个参数是 `foo3`，上面的命令是不会返回 key `foo3` 的值的。具体可以参考 API 说明。

## 读取 key 过往版本的值

应用可能想读取 key 的被替代的值。例如，应用可能想通过访问 key 的过往版本来回滚到旧的配置。或者，应用可能想通过访问 key 历史记录的多个请求来得到一个覆盖多个 key 上的统一视图。

因为 etcd 集群上键值存储的每个修改都会增加 etcd 集群的全局修订版本，应用可以通过提供旧有的 etcd 版本来读取被替代的 key。

假设 etcd 集群已经有下列 key：

```bash
$ etcdctl put foo bar         # revision = 2
$ etcdctl put foo1 bar1       # revision = 3
$ etcdctl put foo bar_new     # revision = 4
$ etcdctl put foo1 bar1_new   # revision = 5
```

这里是访问 key 的过往版本的例子：

```bash
$ etcdctl get foo foo9 # 访问 key 的最新版本
foo
bar_new
foo1
bar1_new

$ etcdctl get --rev=4 foo foo9 # 访问 key 的修订版本4
foo
bar_new
foo1
bar1

$ etcdctl get --rev=3 foo foo9 # 访问 key 的修订版本3
foo
bar
foo1
bar1

$ etcdctl get --rev=2 foo foo9 # 访问 key 的修订版本2
foo
bar

$ etcdctl get --rev=1 foo foo9 # 访问 key 的修订版本1
```

## 删除 key

应用可以从 etcd 集群中删除一个 key 或者特定范围的 key。

下面是删除 key `foo` 的命令：

```bash
$ etcdctl del foo
1 # 删除了一个key
```
这是删除从 `foo` to `foo9` 范围的 key 的命令：

```bash
$ etcdctl del foo foo9
2 # 删除了两个key
```

## 观察 key 的变化

应用可以观察一个 key 或者特定范围内的 key 来监控任何更新。

这是在 key `foo` 上进行观察的命令：

```bash
$ etcdctl watch foo
# 在另外一个终端: etcdctl put foo bar
foo
bar
```

这是观察从 `foo` to `foo9` 范围key的命令：

```bash
$ etcdctl watch foo foo9
# 在另外一个终端: etcdctl put foo bar
foo
bar
# 在另外一个终端: etcdctl put foo1 bar1
foo1
bar1
```

## 观察 key 的历史改动

应用可能想观察 etcd 中 key 的历史改动。例如，应用想接收到某个 key 的所有修改。如果应用一直连接到etcd，那么 `watch` 就足够好了。但是，如果应用或者 etcd 出错，改动可能发生在出错期间，这样应用就没能实时接收到这个更新。为了保证更新被接收，应用必须能够观察到 key 的历史变动。为了做到这点，应用可以在观察时指定一个历史修订版本，就像读取 key 的过往版本一样。

假设我们完成了下列操作序列：

``` bash
etcdctl put foo bar         # revision = 2
etcdctl put foo1 bar1       # revision = 3
etcdctl put foo bar_new     # revision = 4
etcdctl put foo1 bar1_new   # revision = 5
```

这是观察历史改动的例子：

```bash
# 从修订版本 2 开始观察key `foo` 的改动
$ etcdctl watch --rev=2 foo
PUT
foo
bar
PUT
foo
bar_new

# 从修订版本 3 开始观察key `foo` 的改动
$ etcdctl watch --rev=3 foo
PUT
foo
bar_new
```

## 压缩修订版本

如我们提到的，etcd 保存修订版本以便应用可以读取 key 的过往版本。但是，为了避免积累无限数量的历史数据，压缩过往的修订版本就变得很重要。压缩之后，etcd 删除历史修订版本，释放资源来提供未来使用。所有修订版本在压缩修订版本之前的被替代的数据将不可访问。

这是压缩修订版本的命令：

```bash
$ etcdctl compact 5
compacted revision 5

# 在压缩修订版本之前的任何修订版本都不可访问
$ etcdctl get --rev=4 foo
Error:  rpc error: code = 11 desc = etcdserver: mvcc: required revision has been compacted
```

## 授予租约

应用可以为 etcd 集群里面的 key 授予租约。当 key 被附加到租约时，它的生存时间被绑定到租约的生存时间，而租约的生存时间相应的被 `time-to-live` (TTL)管理。租约的实际 TTL 值是不低于最小 TTL，由 etcd 集群选择。一旦租约的 TTL 到期，租约就过期并且所有附带的 key 都将被删除。

这是授予租约的命令：

```bash
# 授予租约，TTL为10秒
$ etcdctl lease grant 10
lease 32695410dcc0ca06 granted with TTL(10s)

# 附加key foo到租约32695410dcc0ca06
$ etcdctl put --lease=32695410dcc0ca06 foo bar
OK
```

## 撤销租约

应用通过租约 id 可以撤销租约。撤销租约将删除所有它附带的 key。

假设我们完成了下列的操作：

```bash
$ etcdctl lease grant 10
lease 32695410dcc0ca06 granted with TTL(10s)
$ etcdctl put --lease=32695410dcc0ca06 foo bar
OK
```

这是撤销同一个租约的命令：

```bash
$ etcdctl lease revoke 32695410dcc0ca06
lease 32695410dcc0ca06 revoked

$ etcdctl get foo
# 空应答，因为租约撤销导致foo被删除
```

## 维持租约

应用可以通过刷新 key 的 TTL 来维持租约，以便租约不过期。

假设我们完成了下列操作：

```bash
$ etcdctl lease grant 10
lease 32695410dcc0ca06 granted with TTL(10s)
```

这是维持同一个租约的命令：

```bash
$ etcdctl lease keep-alive 32695410dcc0ca0
lease 32695410dcc0ca0 keepalived with TTL(100)
lease 32695410dcc0ca0 keepalived with TTL(100)
lease 32695410dcc0ca0 keepalived with TTL(100)
...
```

> 注： 上面的这个命令中，etcdctl 不是单次续约，而是 etcdctl 会一直不断的发送请求来维持这个租约。