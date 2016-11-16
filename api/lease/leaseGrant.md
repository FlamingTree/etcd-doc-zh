# LeaseGrant 方法

LeaseGrant 方法创建一个租约.

```java
rpc LeaseGrant(LeaseGrantRequest) returns (LeaseGrantResponse) {}
```

## 消息体

请求的消息体是 LeaseGrantRequest：

```java
message LeaseGrantRequest {
  // TTL 是建议的以秒为单位的 time-to-live
  int64 TTL = 1;

  // DI 是租约的请求ID。如果ID设置为0, 则出租人(也就是etcd server)选择一个ID。
  int64 ID = 2;
}
```

应答的消息体是 LeaseGrantResponse：

```java
message LeaseGrantResponse {
  ResponseHeader header = 1;
  // ID 是承认的租约的ID
  int64 ID = 2;
  // TTL 是服务器选择的以秒为单位的租约time-to-live
  int64 TTL = 3;
  string error = 4;
}
```

两个待后续继续理解的细节：

1. 请求中的TTL和应答中的TTL： 貌似请求只是给个建议，而服务器可能接收或者给出自己的决定。因此处理时要考虑两者不一致的情况？什么情况服务器会不依从客户端的建议。
2. 应答中的error： 什么情况下会使用？

