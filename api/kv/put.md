# Put 方法

Put方法放置给定key到键值存储.

Put方法增加键值存储的修订版本并在事件历史中生成一个事件.

```java
rpc Put(PutRequest) returns (PutResponse) {}
```

## 消息体

请求的消息体是 PutRequest：

```java
message PutRequest {
  // byte数组形式的key，用来放置到键值对存储
  bytes key = 1;

  // byte数组形式的value，在键值对存储中和key关联
  bytes value = 2;

  // 在键值存储中和key关联的租约id。0代表没有租约。
  int64 lease = 3;

  // 如果 prev_kv 被设置，etcd获取改变之前的上一个键值对。
  // 上一个键值对将在put应答中被返回
  bool prev_kv = 4;
}
```

应答的消息体是 PutResponse：

```java
message PutResponse {
  ResponseHeader header = 1;

  // 如果请求中的 prev_kv 被设置，将会返回上一个键值对
  mvccpb.KeyValue prev_kv = 2;
}
```