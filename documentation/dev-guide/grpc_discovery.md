# gRPC服务发现

etcd 为gRPC提供了服务发现的解决方案。背后的工作机制是 `watch` 以相应服务名做前缀的 key

## 使用 etcd 为 go-grpc 做服务发现

etcd.naming 模块提供了 `reslover` 用于解析 gRPC 服务地址：

```go
import (
    "github.com/coreos/etcd/clientv3"
    etcdnaming "github.com/coreos/etcd/clientv3/naming"

    "google.golang.org/grpc"
)

...

cli, cerr := clientv3.NewFromURL("http://localhost:2379")
r := &etcdnaming.GRPCResolver{Client: cli}
b := grpc.RoundRobin(r)
conn, gerr := grpc.Dial("my-service", grpc.WithBalancer(b))
```

## 服务管理

etcd reslover把以 target+"/"（例如："my-service/"）为前缀的 key 的值作为gRPC服务的地址，值的格式为json编码的`naming.Update`。服务节点通过增加或删除 key 进行服务注册和服务注销。

### 服务注册

新的服务节点可以通过 `etcdctl` 进行注册：

```bash
ETCDCTL_API=3 etcdctl put my-service/1.2.3.4 '{"Addr":"1.2.3.4","Metadata":"..."}'
```

或者通过在代码中通过 `GRPCResolver.Update` 进行注册：

```go
r.Update(context.TODO(), "my-service", naming.Update{Op: naming.Add, Addr: "1.2.3.4", Metadata: "..."})
```

### 服务注销

可以通过 `etcdctl` 注销服务节点：

```bash
ETCDCTL_API=3 etcdctl del my-service/1.2.3.4
```

同样也可以在代码中通过`GRPCResolver.Update` 进行注销：

```go
r.Update(context.TODO(), "my-service", naming.Update{Op: naming.Delete, Addr: "1.2.3.4"})
```

### 通过 lease 注册

通过 lease 注册保证当主机不能进行 keepalive 心跳时（如宕机，网络分区），它将被注销掉：

```bash
lease=`ETCDCTL_API=3 etcdctl lease grant 5 | cut -f2 -d' '`
ETCDCTL_API=3 etcdctl put --lease=$lease my-service/1.2.3.4 '{"Addr":"1.2.3.4","Metadata":"..."}'
ETCDCTL_API=3 etcdctl lease keep-alive $lease
```

