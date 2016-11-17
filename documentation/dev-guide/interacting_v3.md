# 和 etcd 交互

> 注： 内容翻译自 [Interacting with etcd](https://github.com/coreos/etcd/blob/master/Documentation/dev-guide/interacting_v3.md)

用户通常通过设置或者获取 key 的值来和 etcd 交互。这一节描述如何使用 etcdctl 来操作， etcdctl 是一个和 etcd 服务器交互的命令行工具。这里描述的概念也适用于 gRPC API 或者客户端类库 API。

为了向后兼容 etcdctl 默认使用 v2 API 来和 etcd 服务器通讯。为了让 etcdctl 使用 v3 API 来和etcd通讯，API 版本必须通过环境变量 `ETCDCTL_API` 设置为版本3。

``` bash
export ETCDCTL_API=3
```

## 查询版本
通常需要根据 etcdctl 和 API 版本的不同对 etcd 做不同的操作。查询版本的命令：

```bash
$ etcdctl version
etcdctl version: 3.1.0-alpha.0+git
API version: 3.1
```

## 写入key

应用通过写入 key 来储存 key 到 etcd 中。每个存储的 key 通过 Raft 协议复制到所有 etcd 集群成员来达到一致性和可靠性。

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

这是读取 key `foo` 的值的命令：

```bash
$ etcdctl get foo
foo
bar
```

读取值并以16进制返回：

```bash
$ etcdctl get foo --hex
\x66\x6f\x6f          # Key
\x62\x61\x72          # Value
```

这是读取从 `foo` to `foo9` 的 key 的命令：

```bash
$ etcdctl get foo foo9
foo
bar
foo1
bar1
foo3
bar3
```

读取从 `foo` to `foo9` 的 key，至多返回2条：

```bash
$ etcdctl get foo foo9 --limit 2
foo
bar
foo1
bar1
```

其它选项：`--from-key`, `--keys-only`, `--order`, `--sort-by`, `--prefix`
> 注： 测试中发现，这个命令的有效区间是这样：`[foo foo9)`， 即包含第一个参数，但是不包含第二个参数。因此如果第二个参数是 `foo3`，上面的命令是不会返回 key `foo3` 的值的。具体可以参考 API 说明。

## 读取 key 过往版本的值

应用可能想读取 key 的被替代的值。例如，应用可能想通过访问 key 的过往版本来回滚到旧的配置。或者，应用可能想通过多个请求获取 key 的历史记录来得到多个 key 的一致视图。

因为对 etcd 集群上键值存储的每次修改都会增加 etcd 集群的全局修订版本，应用可以读取 key 的旧版本值。

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

## 读取大于或等于指定 key 的键值

应用可能要读取大于或等于指定 key 的键值，假设 etcd 集群存在下列 key:

```
a = 123
b = 456
z = 789
```

获取值大于 `b` 的键：

```bash
$ etcdctl get --from-key b
b
456
z
789
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

删除 `zoo` 键并返回删除的键值：

```bash
$ etcdctl del --prev-kv zoo 
1   # one key is deleted
zoo # deleted key
val # the value of the deleted key
```

删除前缀为 `zoo` 的键：

```bash
$ etcdctl del --prefix zoo 
2 # two keys are deleted
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

观察前缀为 `foo` 的 key：

```bash
$ etcdctl watch --prefix foo
# in another terminal: etcdctl put foo bar
PUT
foo
bar
# in another terminal: etcdctl put fooz1 barz1
PUT
fooz1
barz1
```

观察`foo`键 和 `zoo`键:

```bash
$ etcdctl watch -i 
watch foo
watch zoo
# in another terminal: etcdctl put foo bar
PUT
foo
bar
# in another terminal: etcdctl put zoo val
PUT
zoo
val
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

观察并返回最近一次的修改：

```bash
# 观察 `foo` 的变化，返回最近一次的修改历史和最新值
$ etcdctl watch --prev-kv foo
# in another terminal: etcdctl put foo bar_latest
PUT
foo         # key
bar_new     # 修改之前 foo 的值
foo         # key
bar_latest  # 修改之后 foo 的值
```

## 压缩修订版本

如我们提到的，etcd 保存修订版本以便应用可以读取 key 的过往版本。但是，为了避免历史数据无限增长，压缩过往的修订版本就变得很重要。压缩之后，etcd 删除历史修订版本，释放资源来提供未来使用。所有压缩修订版本之前的版本的数据将不可访问。

这是压缩修订版本的命令：

```bash
$ etcdctl compact 5
compacted revision 5

# 在压缩版本之前的任何版本都不可访问
$ etcdctl get --rev=4 foo
Error:  rpc error: code = 11 desc = etcdserver: mvcc: required revision has been compacted
```

**注意**：当前版本可通过获取 key 的json格式的数据得到，无论 key 是否存在。例如，获取 `mykey` 的值，`mykey` 在 etcd 中并不存在：

```bash
$ etcdctl get mykey -w=json
{"header":{"cluster_id":14841639068965178418,"member_id":10276657743932975437,"revision":15,"raft_term":4}}
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

应用可以通过刷新 key 的 TTL 来保活（keep alive）租约，以便租约不过期。

假设我们完成了下列操作：

```bash
$ etcdctl lease grant 10
lease 32695410dcc0ca06 granted with TTL(10s)
```

这是保活同一个租约的命令：

```bash
$ etcdctl lease keep-alive 32695410dcc0ca0
lease 32695410dcc0ca0 keepalived with TTL(100)
lease 32695410dcc0ca0 keepalived with TTL(100)
lease 32695410dcc0ca0 keepalived with TTL(100)
...
```

> 注： 上面的这个命令中，etcdctl 不是单次续约，而是 etcdctl 会一直不断的发送请求来维持这个租约。

## 获取租约信息

应用可能想获取租约信息，这样就能更新租约或者检查租约是否存在或过期。应用可能还想获取租约关联了哪些 key。

假设进行了以下操作：

```bash
# grant a lease with 500 second TTL
$ etcdctl lease grant 500
lease 694d5765fc71500b granted with TTL(500s)

# attach key zoo1 to lease 694d5765fc71500b
$ etcdctl put zoo1 val1 --lease=694d5765fc71500b
OK

# attach key zoo2 to lease 694d5765fc71500b
$ etcdctl put zoo2 val2 --lease=694d5765fc71500b
OK
```

获取租约信息：

```bash
$ etcdctl lease timetolive 694d5765fc71500b
lease 694d5765fc71500b granted with TTL(500s), remaining(258s)
```

获取租约关联的 key：

```bash
$ etcdctl lease timetolive --keys 694d5765fc71500b
lease 694d5765fc71500b granted with TTL(500s), remaining(132s), attached keys([zoo2 zoo1])

# if the lease has expired or does not exist it will give the below response:
Error:  etcdserver: requested lease not found
```

