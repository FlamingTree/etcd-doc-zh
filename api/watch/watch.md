# Watch 方法

WatchService 只有一个 Watch 方法。

## 消息定义

请求的消息体是 WatchRequest：

```java

message WatchRequest {
  // request_union 要么是创建新的观察者的请求，要么是取消一个已经存在的观察者的请求
  oneof request_union {
    WatchCreateRequest create_request = 1;
    WatchCancelRequest cancel_request = 2;
  }
}
```

创建新的观察者的请求 WatchCreateRequest：

```java
message WatchCreateRequest {
  // key 是注册要观察的key
  bytes key = 1;

  // range_end 是要观察的范围 [key, range_end) 的终点。
  // 如果 range_end 没有设置，则只有参数key被观察。
  // 如果 range_end 等同于 '\0'， 则大于等于参数key的所有key都将被观察
  bytes range_end = 2;

  // start_revision 是可选的开始(包括)观察的修订版本。不设置 start_revision 则表示 "现在".
  int64 start_revision = 3;

  // 设置 progress_notify ， 这样如果最近没有事件，etcd 服务器将定期的发送不带任何事件的 WatchResponse 给新的观察者。
  // 当客户端希望从最近已知的修订版本开始恢复断开的观察者时有用。
  // etcd服务器将基于当前负载决定它发送通知的频率。
  bool progress_notify = 4;

  enum FilterType {
  // 过滤掉 put 事件
  NOPUT = 0;

  // 过滤掉 delete 事件
  NODELETE = 1;
  }

  // 过滤器，在服务器端发送事件给回观察者之前，过滤掉事件。
  repeated FilterType filters = 5;

  // 如果 prev_kv 被设置，被创建的观察者在事件发生前获取上一次的KV。
  // 如果上一次的KV已经被压缩，则不会返回任何东西
  bool prev_kv = 6;
}
```

取消已有观察者的 WatchCancelRequest ：

```java
message WatchCancelRequest {
  // watch_id 是要取消的观察者的id，这样就不再有更多事件传播过来了。
  int64 watch_id = 1;
}
```

应答的消息体 WatchResponse：

```java
message WatchResponse {
  ResponseHeader header = 1;
  // watch_id 是和应答相关的观察者的ID
  int64 watch_id = 2;

  // 如果应答是用于创建观察者请求的，则 created 设置为true。
  // 客户端应该记录 watch_id 并期待从同样的流中为创建的观察者接收事件。
  // 所有发送给被创建的观察者的事件将附带同样的watch_id
  bool created = 3;

  // 如果应答是用于取消观察者请求的，则 canceled 设置为true。
  // 不会再有事件发送给被取消的观察者。
  bool canceled = 4;

  // compact_revision 被设置为最小index，如果观察者试图观察被压缩的index。
  // 当在被压缩的修订版本上创建观察者或者观察者无法追上键值对存储的进展时发生。
  // 客户端应该视观察者为被取消，并不应该试图再次创建任何带有相同 start_revision 的观察者。
  int64 compact_revision  = 5;

  repeated mvccpb.Event events = 11;
}
```

mvccpb.Event 的消息体：

```java
message Event {
  enum EventType {
    PUT = 0;
    DELETE = 1;
  }

  // type是事件的类型。
  // 如果类型是 PUT，表明新的数据已经存储到key。
  // 如果类型是 DELETE， 表明key已经被删除。
  EventType type = 1;

  // kv 为事件持有KeyValue。
  // PUT 事件包含当前的kv键值对
  // DELETE/EXPIRE 事件包含被删除的key，它的修改修订版本设置为删除的修订版本
  KeyValue kv = 2;

  // prev_kv 持有在事件发生前的键值对
  KeyValue prev_kv = 3;
}
```


