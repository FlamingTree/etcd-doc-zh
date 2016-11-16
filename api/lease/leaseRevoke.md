# LeaseGrant 方法

LeaseGrant 方法取消一个租约.

```java
rpc LeaseRevoke(LeaseRevokeRequest) returns (LeaseRevokeResponse) {}
```

## 消息体

请求的消息体是 LeaseRevokeRequest：

```java
message LeaseRevokeRequest {
  // ID是要取消的租约的ID。
  // 当租约被取消时，所有关联的key将被删除
  int64 ID = 1;
}
```

应答的消息体是 LeaseRevokeResponse：

```java
message LeaseRevokeResponse {
  ResponseHeader header = 1;
}
```


