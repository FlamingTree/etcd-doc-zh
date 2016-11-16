# Range方法

Range方法从键值存储中获取范围内的key.

```java
rpc Range(RangeRequest) returns (RangeResponse) {}
```

注意没有操作单个key的方法，即使是存取单个key，也是需要使用 `Range` 方法的。

## 消息体

请求的消息体是RangeRequest：

```java
message RangeRequest {
  enum SortOrder {
	NONE = 0; // 默认, 不排序
	ASCEND = 1; // 正序，低的值在前
	DESCEND = 2; // 倒序，高的值在前
  }
  enum SortTarget {
	KEY = 0;
	VERSION = 1;
	CREATE = 2;
	MOD = 3;
	VALUE = 4;
  }

  // key是range的第一个key。如果 range_end 没有给定，请求仅查找这个key
  bytes key = 1;

  // range_end 是请求范围的上限[key, range_end)
  // 如果 range_end 是 '\0'，范围是大于等于 key 的所有key。
  // 如果 range_end 比给定的key长一个bit， 那么范围请求获取所有带有前缀(给定的key)的key
  // 如果 key 和 range_end 都是'\0'，则范围查询返回所有key
  bytes range_end = 2;

  // 请求返回的key的数量限制
  int64 limit = 3;

  // 修订版本是用于范围的键值对存储的时间点。
  // 如果 revision 小于或等于零，范围是在最新的键值对存储上。
  // 如果修订版本已经被压缩，返回 ErrCompacted 作为应答
  int64 revision = 4;

  // 指定返回结果的排序顺序
  SortOrder sort_order = 5;

  // 用于排序的键值字段
  SortTarget sort_target = 6;

  // 设置范围请求使用串行化成员本地读(serializable member-local read)。
  // 范围请求默认是线性化的;线性化请求相比串行化请求有更高的延迟和低吞吐量，但是反映集群当前的一致性。
  // 为了更好的性能，以可能脏读为交换，串行化范围请求在本地处理，无需和集群中的其他节点达到一致。
  bool serializable = 7;

  // 设置仅返回key而不需要value
  bool keys_only = 8;

  // 设置仅仅返回范围内key的数量
  bool count_only = 9;
}
```

应答的消息体是 RangeResponse：

```java
message RangeResponse {
  ResponseHeader header = 1;

  // kvs是匹配范围请求的键值对列表
  // 当请求数量时是空的
  repeated mvccpb.KeyValue kvs = 2;

  // more代表在被请求的范围内是否还有更多的key
  bool more = 3;

  // 被请求范围内key的数量
  int64 count = 4;
}
```

mvccpb.KeyValue 的消息体：

```java
message KeyValue {
  // key是bytes格式的key。不容许key为空。
  bytes key = 1;

  // create_revision 是这个key最后创建的修订版本
  int64 create_revision = 2;

  // mod_revision 是这个key最后修改的修订版本
  int64 mod_revision = 3;

  // version 是key的版本。删除会重置版本为0,而任何key的修改会增加它的版本。
  int64 version = 4;

  // value是key持有的值，bytes格式。
  bytes value = 5;

  // lease是附加给key的租约id。
  // 当附加的租约过期时，key将被删除。
  // 如果lease为0,则没有租约附加到key。
  int64 lease = 6;
}
```

