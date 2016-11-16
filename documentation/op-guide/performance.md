# 性能

> 注：内容翻译自 [Performance](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/performance.md)


## 理解性能

etcd 提供稳定的，持续的高性能。两个定义性能的因素：延迟(latency)和吞吐量(throughput)。延迟是完成操作的时间。吞吐量是在某个时间期间之内完成操作的总数量。当 etcd 接收并发客户端请求时，通常平均延迟随着总体吞吐量增加而增加。在通常的云环境，比如 Google Compute Engine (GCE) 标准的 `n-4` 或者 AWS 上相当的机器类型，一个三成员 etcd 集群在轻负载下可以在低于1毫秒内完成一个请求，并在重负载下可以每秒完成超过 30000 个请求。

etcd 使用 Raft 一致性算法来在成员之间复制请求并达成一致。一致性性能，特别是提交延迟，受限于两个物理约束：网络IO延迟和磁盘IO延迟。完成一个etcd请求的最小时间是成员之间的网络往返时延(Round Trip Time / RTT)，加需要提交数据到持久化存储的 `fdatasync` 时间。在一个数据中心内的 RTT 可能有数百毫秒。在美国典型的 RTT 是大概 50ms, 而在大陆之间可以慢到400ms. 旋转硬盘(注：指传统机械硬盘)的典型 fdatasync 延迟是大概 10ms。对于 SSD 硬盘, 延迟通常低于 1ms. 为了提高吞吐量, etcd 将多个请求打包在一起并提交给 Raft。这个批量策略让 etcd 在重负载试获得高吞吐量.

有其他子系统影响到 etcd 的整体性能。每个序列化的 etcd 请求必须通过 etcd 的 boltdb支持的(boltdb-backed) MVCC 存储引擎,它通常需要10微秒来完成。etcd 定期递增快照它最近实施的请求，将他们和之前在磁盘上的快照合并。这个过程可能导致延迟尖峰(latency spike)。虽然在SSD上这通常不是问题，在HDD上它可能加倍可观察到的延迟。而且，进行中的压缩可以影响 etcd 的性能。幸运的是，压缩通常无足轻重，因为压缩是错开的，因此它不和常规请求竞争资源。RPC 系统，gRPC，为 etcd 提供定义良好，可扩展的 API，但是它也引入了额外的延迟，尤其是本地读取。

## 评测

可以通过 etcd 自带的 [benchmark](https://github.com/coreos/etcd/tree/master/tools/benchmark) CLI 工具来评测 etcd 的性能.

对于某些基线性能数字，我们考虑的是3成员 etcd 集群，带有下列硬件配置：

- Google Cloud Compute Engine
- 3 台机器， 8 vCPUs + 16GB Memory + 50GB SSD
- 1 台机器(客户端)，16 vCPUs + 30GB Memory + 50GB SSD
- Ubuntu 15.10
- etcd v3 master 分支 (commit SHA d8f325d), Go 1.6.2

使用这些配置，etcd 能近似的写入：

| key的数量 | Key的大小 | Value的大小 | 连接数量 | 客户端数量 | 目标 etcd 服务器 | 平均写入 QPS | 每请求平均延迟 | 内存 |
|----------------|-------------------|---------------------|-----------------------|-------------------|--------------------|-------------------|-----------------------------|--------|
| 10,000 | 8 | 256 | 1 | 1 | leader only | 525 | 2ms | 35 MB |
| 100,000 | 8 | 256 | 100 | 1000 | leader only | 25,000 | 30ms | 35 MB |
| 100,000 | 8 | 256 | 100 | 1000 | all members | 33,000 | 25ms | 35 MB |

> 注：key和value的大小单位是 字节 / bytes

采样命令是:

```bash
# 假定 IP_1 是 leader, 写入请求发到 leader
benchmark --endpoints={IP_1} --conns=1 --clients=1 \
    put --key-size=8 --sequential-keys --total=10000 --val-size=256
benchmark --endpoints={IP_1} --conns=100 --clients=1000 \
    put --key-size=8 --sequential-keys --total=100000 --val-size=256

# 写入发到所有成员
benchmark --endpoints={IP_1},{IP_2},{IP_3} --conns=100 --clients=1000 \
    put --key-size=8 --sequential-keys --total=100000 --val-size=256
```

为了一致性，线性化(Linearizable)读取请求要通过集群成员的法定人数来获取最新的数据。串行化(Serializable)读取请求比线性化读取要廉价一些，因为他们是通过任意单台 etcd 服务器来提供服务，而不是成员的法定人数，代价是可能提供过期数据。etcd 可以读取：

| 请求数量 | Key 大小 | Value 大小 | 连接数量 | 客户端数量 | 一致性 | 每请求平均延迟 | 平均读取 QPS |
|--------------------|-------------------|---------------------|-----------------------|-------------------|-------------|-----------------------------|------------------|
| 10,000 | 8 | 256 | 1 | 1 | Linearizable | 2ms | 560 |
| 10,000 | 8 | 256 | 1 | 1 | Serializable | 0.4ms | 7,500 |
| 100,000 | 8 | 256 | 100 | 1000 | Linearizable | 15ms | 43,000 |
| 100,000 | 8 | 256 | 100 | 1000 | Serializable | 9ms | 93,000 |

采样命令是:

```bash
# Linearizable 读取请求
benchmark --endpoints={IP_1},{IP_2},{IP_3} --conns=1 --clients=1 \
    range YOUR_KEY --consistency=l --total=10000
benchmark --endpoints={IP_1},{IP_2},{IP_3} --conns=100 --clients=1000 \
    range YOUR_KEY --consistency=l --total=100000

# Serializable 读取请求，使用每个成员然后将数字加起来
for endpoint in {IP_1} {IP_2} {IP_3}; do
    benchmark --endpoints=$endpoint --conns=1 --clients=1 \
        range YOUR_KEY --consistency=s --total=10000
done
for endpoint in {IP_1} {IP_2} {IP_3}; do
    benchmark --endpoints=$endpoint --conns=100 --clients=1000 \
        range YOUR_KEY --consistency=s --total=100000
done
```

当在新的环境中第一次搭建 etcd 集群时，我们鼓励运行评测测试来保证集群达到足够的性能;集群延迟和吞吐量对微小的环境差异会很敏感。

