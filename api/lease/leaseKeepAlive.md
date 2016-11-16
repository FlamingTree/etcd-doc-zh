# LeaseKeepAlive 方法

LeaseKeepAlive 方法取消一个租约.

```java
rpc LeaseKeepAlive(stream LeaseKeepAliveRequest) returns (stream LeaseKeepAliveResponse) {}
```

## 消息体

请求的消息体是 LeaseKeepAliveRequest：

```java
message LeaseKeepAliveRequest {
  // ID是要继续存活的租约的ID
  int64 ID = 1;
}
```

应答的消息体是 LeaseKeepAliveResponse：

```java
message LeaseKeepAliveResponse {
  ResponseHeader header = 1;
  // ID是从继续存活请求中得来的租约ID
  int64 ID = 2;
  // TTL是租约新的 time-to-live
  int64 TTL = 3;
}
```


