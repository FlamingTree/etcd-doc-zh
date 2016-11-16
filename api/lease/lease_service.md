# Lease service

Lease service提供租约的支持。

在 rpc.proto 文件中 Lease service 定义如下：

```java
service Lease {
  // LeaseGrant 创建一个租约，当服务器在给定时间内没有接收到 keepAlive 时租约过期。
  // 如果租约过期则所有附加在租约上的key将过期并被删除。
  // 每个过期的key在事件历史中生成一个删除事件。
  rpc LeaseGrant(LeaseGrantRequest) returns (LeaseGrantResponse) {}

  // LeaseRevoke 撤销一个租约。
  // 所有附加到租约的key将过期并被删除。
  rpc LeaseRevoke(LeaseRevokeRequest) returns (LeaseRevokeResponse) {}

  // LeaseKeepAlive 通过从客户端到服务器端的流化的keep alive 请求和从服务器端到客户端的流化的keep alive应答来维持租约.
  rpc LeaseKeepAlive(stream LeaseKeepAliveRequest) returns (stream LeaseKeepAliveResponse) {}
}
```

