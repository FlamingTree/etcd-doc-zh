# Lease service

Lease service提供租约的支持。

在 rpc.proto 文件中 Lease service 定义如下：

```grpc
service Lease {
  // LeaseGrant 创建一个租约，当服务器在给定时间内没有接收到 keepAlive 时租约过期。
  // 如果租约过期则所有附加在租约上的key将过期并被删除。
  // 每个过期的key在事件历史中生成一个删除事件。
  rpc LeaseGrant(LeaseGrantRequest) returns (LeaseGrantResponse) {}

  // LeaseRevoke 撤销一个租约。
  // 所有附加到租约的key将过期并被删除。
  rpc LeaseRevoke(LeaseRevokeRequest) returns (LeaseRevokeResponse) {}

  // LeaseKeepAlive 通过客户端持续向服务器端发送 keepalive 请求和不断接收从服务端返回的 keepalive应答来维持租约。
  rpc LeaseKeepAlive(stream LeaseKeepAliveRequest) returns (stream LeaseKeepAliveResponse) {}
}
```

