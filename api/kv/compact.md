# Compact 方法

Compact 方法压缩etcd键值存储中的事件历史。

键值存储应该定期压缩，否则事件历史会无限增长.

```grpc
rpc Compact(CompactionRequest) returns (CompactionResponse) {}
```

## 消息体

请求的消息体是 PutRequest：

```grpc
message CompactionRequest {
  // 用于压缩操作的键值存储修订版本号
  int64 revision = 1;

  // physical设置为 true 时 RPC 将会等待直到压缩在落地到物理存储，导致压缩的项完全从物理存储中移除
  bool physical = 2;
}
```

应答的消息体是 PutResponse：

```grpc
message CompactionResponse {
  ResponseHeader header = 1;
}
```


