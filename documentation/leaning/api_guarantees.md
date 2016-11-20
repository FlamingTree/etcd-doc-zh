# KV API 保证

> 注： 内容翻译自 [KV API grarentees](https://github.com/coreos/etcd/blob/master/Documentation/learning/api_guarantees.md)
>
> 注2： 在官网文档页面中没有看到这个文档的链接，我是在git仓库的leaning目录下找到的。

etcd是一致而持久的键值存储，带有小型事务(mini-transaction)支持。键值对存储暴露给KV API。etcd试图为分布式系统提供强一致性和持久性的支持。这份规范列举 etcd 实现的 KV API 保证。

### 涉及的 API

* 读 API
    * range
    * watch
* 写 API
    * put
    * delete
* 组合 (读-改-写) API
    * txn

### etcd 详细的定义

#### 操作完成

当etcd操作通过一致行协议提交，因此被存储引擎“执行”（持久化存储）后，操作视为成功。当客户端从 etcd 服务器接收到应答时，客户端得知操作已经完成。注意，如果客户端和etcd成员之间有超时或者网络中断，客户端可能对操作的状态不确定。当有 leader 选举时，etcd 也可能中止操作。在这种情况下，etcd不会发送 `abort` 应答给客户端的未完成的请求。

#### Revision / 修订版本

修改键值存储的 etcd 操作被分配一个唯一的递增的修订版本。事务操作可能多次修改键值存储，但是只分配一个修订版本。被操作修改的键值对的 revision 值和操作的 revision 值相同。修订版本可以作为键值存储的逻辑时钟。版本较大的键值对一定在版本较小的键值对之后被修改。两个拥有相同修订版本的键值对是被一个操作同时修改的。

### 提供的保证

#### 原子性

所有 API 请求都是原子的; 操作或者成功，或者完全失败。对于 watch 请求，一个操作生成的所有事件将在一个 watch 应答中被返回。watch 绝不会观察到一个操作的部分事件。

#### 一致性

所有API调用确保 [顺序一致性](https://en.wikipedia.org/wiki/Consistency_model#Sequential_consistency), 在分布式系统可用的一致性中最强的。不管客户端的请求发到哪个 etcd 成员，客户端会以相同的顺序读取到相同的事件。如果两个成员完成同样数量的操作，这两个成员的状态是一致的。

对于 watch 操作，etcd 保证所有成员对于相同版本相同键返回相同的值。对于范围操作，etcd有类似的 [线性一致性](#线性一致性) 访问保证；连续访问可能落后于法定人数状态(quorum state)，因此靠后的修订版本还不可用。

> 这段好晦涩，原文：For range operations, etcd has a similar guarantee for linearized access; serialized access may be behind the quorum state, so that the later revision is not yet available.

和所有分布式系统一样，对 etcd 来说确保强一致性是不可能的。etcd 不保证读操作返回的值是所有集群成员中“最新的”("most recent")。

> 叹息一声，翻译不好，原文：etcd does not guarantee that it will return to a read the “most recent” value (as measured by a wall clock when a request is completed) available on any cluster member.

#### 隔离性

etcd确保串行化隔离，这在分布式系统可用的隔离级别中是最高的。读操作不会获取到中间数据。

#### 持久性

任何成功的操作都是持久的。所有可访问的数据也都是持久的数据。读操作从来不会返回没有持久化的数据。

#### 线性一致性

线性一致性（也被称为 Atomic Consistency 或者 External Consistency）是介于 strict consistency/严格一致性和 sequential consistency/顺序一致性之间的一致性级别。

对于线性一致性，假设每个操作从未严格同步的全局时钟获取时间戳。操作是线性化的当且仅当，操作是按顺序执行的且操作完成的顺序和程序制定的一致。同样的，如果一个操作的时间戳在另外一个之前，操作执行也必须靠前。

> 这句话翻译的好痛苦，还是看原文吧: Operations are linearized if and only if they always complete as though they were executed in a sequential order and each operation appears to complete in the order specified by the program.

例如，考虑客户端在时间点1(*t1*)完成一个写操作。客户端在*t2* (*t2* > *t1*)发送的读请求的结果应该至少是在*t1*时间点写操作完成之后的值。然后，读可能实际在*t3*完成，而返回在*t2*时间的值，可能在*t3*时刻已经是旧值。

对于 watch 操作 etcd 不保证线性一致性。期望用户验证 watch 应答的修订版本来确保正确顺序。

默认地，etcd 对于所有其他操作保证线性一致性。无论如何线性一致性会带来开销，因为线性化请求必须进过 Raft 一致性处理。为了使读请求获得更低的延迟和更高的吞吐量，客户端可以配置请求的一致性模型为 `serializable`，这可能访问到旧的数据（相对于多数成员来说），但是消除了线性化访问时对一致性依赖带来的性能损耗。

