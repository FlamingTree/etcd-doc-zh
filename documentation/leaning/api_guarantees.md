# KV API 保证

> 注： 内容翻译自 [KV API grarentees](https://github.com/coreos/etcd/blob/master/Documentation/leaning/api_guarantees.md)
>
> 注2： 在官网文档页面中没有看到这个文档的链接，我是在git仓库的leaning目录下找到的。

etcd是一致而持久的键值存储，带有微事务(mini-transaction)。键值对存储通过KV API暴露。etcd力图为分布式系统获取最强的一致性和持久性保证。这份规范列举etcd实现的 KV API 保证。

### 考虑的 APIs

* 读 APIs
    * range
    * watch
* 写 APIs
    * put
    * delete
* 联合 (读-改-写) APIs
    * txn

### etcd 明确定义

#### 操作完成

当etcd操作通过一致同意而提交时，视为操作完成，并且因此"执行过" -- 永久保存  -- 被etcd存储引擎。当客户端从etcd服务器接收到应答时，客户端得知操作已经完成。注意，如果客户端和etcd成员之间有超时或者网络中断，客户端可能不确定操作的状态。当有leader选举时，etcd也可能中止操作。在这个事件中，etcd不会发送 `abort` 应答给客户端的未完成的请求。

#### revision/修订版本

修改键值存储的etcd操作被分配有一个唯一的增加的修订版本。事务操作可能多次修改键值存储，但是只分配一个修订版本。被操作修改的键值对的revision属性和操作的revision有同样的值。修订版本可以用来给键值存储做逻辑时间。有较大修订版本的键值对在有较小修订版本的键值对之后修改。两个拥有相同修订版本的键值对是被一个操作同时修改。

### 提供的保证

#### 原子性

所有API请求都是原子的; 操作或者完整的完成，或者完全不。对于观察请求，一个操作生成的所有事件将在一个观察应答中。观察绝不会看到一个操作的部分事件。

#### 一致性

所有API调用确保 [顺序一致性](https://en.wikipedia.org/wiki/Consistency_model#Sequential_consistency), 分布式系统可用的最强的一致性保证。不管客户端的请求发往哪个etcd成员,客户端以同样的顺序读取相同的事件。如果两个成员完成同样数量的操作，这两个成员的状态是一致的。

对于观察操作，etcd保证所有成员对于同样的修订版本返回同样的key返回同样的值。对于范围操作，etcd有类似的线性一致性访问保证;连续访问可能落后于法定人数状态(quorum state)，因此后来的修订版本还不用。

> 这段好晦涩，原文：For range operations, etcd has a similar guarantee for linearized access; serialized access may be behind the quorum state, so that the later revision is not yet available.

和所有分布式系统一样，对etcd来说确保强一致性是不可能的。etcd不保证它会给读操作返回在任意集群成员上可以访问的"最近"(most recent)()的值

> 叹息一声，翻译不好，原文：etcd does not guarantee that it will return to a read the “most recent” value (as measured by a wall clock when a request is completed) available on any cluster member.

#### 隔离

etcd确保串行化隔离，这是在分布式系统中可用的最高隔离级别。读操作从不查看任何中间数据。

#### 持久性

任何完成的操作都是持久的。所有可访问的数据也都是持久的数据。读从来不会返回没有持久化的数据。

#### 线性一致性

线性一致性（也被称为 Atomic Consistency 或者 External Consistency）是介于 strict consistency/严格一致性和 sequential consistency/顺序一致性之间的一致性级别。

对于线性一致性，假设每个操作从宽松的已同步全局时钟获取时间戳。如果并且仅当操作总是完成操作是线性化的

操作是线性的，如果并且仅当操作总是完成的好像他们是按顺序执行并且每个操作看上去以程序指定的顺序完成那样。

> 这句话翻译的好痛苦，还是看原文吧: Operations are linearized if and only if they always complete as though they were executed in a sequential order and each operation appears to complete in the order specified by the program.

同样的，如果操作的时间戳发生在其他操作前，这个操作在顺序上必须也发生在其他操作前。

例如，考虑客户端在时间点1(*t1*)完成一个写操作。客户端在*t2* (*t2* > *t1*)发送的读请求应该收到至少在上一次在*t1*时间点完成的写之后最近的值。然后，读可能实际在*t3*完成，而返回值，在*t2*时刻开始读，可能在*t3*时刻已经不再新鲜。

对于观察操作etcd不保证线性一致性。期望用户验证观察应答的修订版本来确保正确顺序。

对于所有其他操作etcd默认保证线性一致性。线性一致性会带来开销，无论如何，因为线性化请求必须通过Raft一致性过程。为了获得读请求更低的延迟和更高的吞吐量，客户端可以配置请求的一致性模型为 `serializable` ,这可能访问不新鲜的数据(with respect to quorum)，但是移除了线性化访问时依赖活动一致性的性能损失。

