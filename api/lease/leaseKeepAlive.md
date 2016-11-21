# LeaseKeepAlive 方法

LeaseKeepAlive 持续发送 keepalive 请求保持租约存活。

```grpc
rpc LeaseKeepAlive(stream LeaseKeepAliveRequest) returns (stream LeaseKeepAliveResponse) {}
```

## 消息体

请求的消息体是 LeaseKeepAliveRequest：

```grpc
message LeaseKeepAliveRequest {
  // ID是要保持存活的租约的ID
  int64 ID = 1;
}
```

应答的消息体是 LeaseKeepAliveResponse：

```grpc
message LeaseKeepAliveResponse {
  ResponseHeader header = 1;
  // ID是 keepalive 请求的租约ID
  int64 ID = 2;
  // TTL是租约新的 TTL
  int64 TTL = 3;
}
```


