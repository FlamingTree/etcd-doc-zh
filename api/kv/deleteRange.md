# DeleteRange 方法

DeleteRange 方法从键值存储中删除给定范围。

删除请求增加键值存储的修订版本并在事件历史中为每个被删除的key生成一个删除事件.

```java
rpc DeleteRange(DeleteRangeRequest) returns (DeleteRangeResponse)
```

## 消息体

请求的消息体是 DeleteRangeRequest：

```java
message DeleteRangeRequest {
  // key是要删除的范围的第一个key
  bytes key = 1;

  // range_end 是要删除范围[key, range_end)的最后一个key
  // 如果 range_end 没有给定，范围定义为仅包含key参数
  // 如果 range_end 是 '\0'， 范围是所有大于等于参数key的所有key。
  bytes range_end = 2;

  // 如果 prev_kv 被设置，etcd获取删除之前的上一个键值对。
  // 上一个键值对将在delete应答中被返回
  bool prev_kv = 3;
}
```

应答的消息体是 DeleteRangeResponse：

```java
message DeleteRangeResponse {
  ResponseHeader header = 1;

  // 被范围删除请求删除的key的数量
  int64 deleted = 2;

  // 如果请求中的 prev_kv 被设置，将会返回上一个键值对
  repeated mvccpb.KeyValue prev_kvs = 3;
}
```