# KV service

KV service提供对键值对操作的支持。

在 rpc.proto 文件中 KV service 定义如下：

```java
service KV {
  // 从键值存储中获取范围内的key.
  rpc Range(RangeRequest) returns (RangeResponse) {}

  // 放置给定key到键值存储.
  // put请求增加键值存储的修订版本并在事件历史中生成一个事件.
  rpc Put(PutRequest) returns (PutResponse) {}

  // 从键值存储中删除给定范围。
  // 删除请求增加键值存储的修订版本并在事件历史中为每个被删除的key生成一个删除事件.
  rpc DeleteRange(DeleteRangeRequest) returns (DeleteRangeResponse) {}

  // 在单个事务中处理多个请求。
  // 一个 txn 请求增加键值存储的修订版本并为每个完成的请求生成带有相同修订版本的事件。
  // 不容许在一个txn中多次修改同一个key.
  rpc Txn(TxnRequest) returns (TxnResponse) {}

  // 压缩在etcd键值存储中的事件历史。
  // 键值存储应该定期压缩，否则事件历史会无限制的持续增长.
  rpc Compact(CompactionRequest) returns (CompactionResponse) {}
}
```






