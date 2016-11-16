# Compact 方法

Compact 方法压缩在etcd键值存储中的事件历史。

键值存储应该定期压缩，否则事件历史会无限制的持续增长.

```java
rpc Compact(CompactionRequest) returns (CompactionResponse) {}
```

## 消息体

请求的消息体是 PutRequest：

```java
message CompactionRequest {
  // 用于比较操作的键值存储的修订版本
  int64 revision = 1;

  // physical设置为 true 时 RPC 将会等待知道压缩物理性的应用到本地数据库，到这程度被压缩的项将完全从后端数据库中移除。
  bool physical = 2;
}
```

应答的消息体是 PutResponse：

```java
message CompactionResponse {
  ResponseHeader header = 1;
}
```
