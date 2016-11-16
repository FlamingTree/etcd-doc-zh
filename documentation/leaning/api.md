# etcd3 API

> 注: 内容翻译自 [etcd3 API](https://github.com/coreos/etcd/blob/master/Documentation/learning/api.md)

注意: 这个文档还没有完成!

> 注：原文如此，的确是还没有完成 :)

## Response header

从etcd API返回的所有应答都附带有 [response header](https://github.com/coreos/etcd/blob/master/etcdserver/etcdserverpb/rpc.proto)。这个response header包含应答的元数据。

```bash
message ResponseHeader {
  uint64 cluster_id = 1;
  uint64 member_id = 2;
  int64 revision = 3;
  uint64 raft_term = 4;
}
```

* Cluster_ID - 生成应答的集群的ID
* Member_ID - 生成应答的成员的ID
* Revision - 当应答生成时键值存储的修订版本
* Raft_Term - 当应答生成时成员的 Raft term

应用可以读取 Cluster_ID (Member_ID) 字段来确保它正在和预期的集群(成员)通讯。

应用可以使用 `Revision/修订版本` 来获知键值存储最新的修订版本。当应用指定一个历史修订版本来实现 `time travel query` 并希望知道请求时刻最新的修订版本时有用。

应用可以使用 `Raft_Term` 来检测集群何时完成了新的leader选举。

## 键值 API

键值API用于操作etcd中的键值对存储。键值API被定义为 [gRPC服务](https://github.com/coreos/etcd/blob/master/etcdserver/etcdserverpb/rpc.proto)。在 [protobuf 格式](https://github.com/coreos/etcd/blob/master/mvcc/mvccpb/kv.proto)中键值对被定义为结构化的数据。

### 键值对

键值对是键值API可以操作的最小单元。每个键值对有一些字段：


```java
message KeyValue {
  bytes key = 1;
  int64 create_revision = 2;
  int64 mod_revision = 3;
  int64 version = 4;
  bytes value = 5;
  int64 lease = 6;
}
```

* Key - 字节数组形式的key。key不容许空。
* Value - 字节数组形式的value
* Version - key的版本。删除将重置版本为0而key的任何修改将增加它的版本。
* Create_Revision - key最后一次创建的修订版本。
* Mod_Revision - key最后一次修改的修订版本。
* Lease - 附加到key的租约的ID。如果lease为0,则表示没有租约附加到key。


