# 配置标记

> 注： 内容翻译自 [Configuration flags](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/configuration.md)

etcd可以通过命令行标记和环境变量来配置。命令行上设置的选项优先级高于环境变量。

对于标记 `--my-flag` 环境变量的格式是 `ETCD_MY_FLAG`。 适用于所有标记。

[正式的ectd端口](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml?search=etcd) 是 2379 用于客户端连接，而 2380 用于节点间通讯。etcd 端口可以设置为接受 TLS 通讯，non-TLS 通讯，或者同时设置 TLS 和 non-TLS 通讯。

为了在 linux 启动时自动启动 etcd ，强烈推荐使用 [systemd](http://freedesktop.org/wiki/Software/systemd/)。

## 成员标记

### --name
+ 成员的易读的名字
+ 默认: "default"
+ 环境变量: ETCD_NAME
+ 这个值被引用为节点的入口， 在 `--initial-cluster` 标记(例如, `default=http://localhost:2380`)中引用。如果使用 [static bootstrapping](clustering.md#静态)，需要标记中使用的key相匹配。当使用 discovery 时，每个成员必须有唯一名字。`Hostname` 或 `machine-id` 是很好的选择。

### --data-dir
+ 数据目录的路径
+ 默认: "${name}.etcd"
+ 环境变量: ETCD_DATA_DIR

### --wal-dir
+ 专用的 wal 目录的路径。如果这个标记被设置，etcd将把 WAL 文件写入到 walDIR 而不是 dataDIR。这容许我们使用专用硬盘，避免与日志和其他IO操作之间的竞争。
+ 默认: ""
+ 环境变量: ETCD_WAL_DIR

### --snapshot-count
+ 触发快照到硬盘的已提交事务的数量
+ 默认: "10000"
+ 环境变量: ETCD_SNAPSHOT_COUNT

### --heartbeat-interval
+ 心跳间隔时间 (单位 毫秒).
+ 默认: "100"
+ 环境变量: ETCD_HEARTBEAT_INTERVAL

### --election-timeout
+ 选举的超时时间(单位 毫秒). 阅读 [Documentation/tuning.md](https://github.com/coreos/etcd/blob/master/Documentation/tuning.md#time-parameters) 获取详情.
+ 默认: "1000"
+ 环境变量: ETCD_ELECTION_TIMEOUT

### --listen-peer-urls

用于监听节点间通讯的URL列表。这个标记告诉 etcd 在指定的 scheme://IP:port 上从接受其他节点的请求。scheme 可是 http 或者 https。如果IP被指定为0.0.0.0，etcd 将在所有接口上监听给定端口。如果指定IP地址和端口，etcd 将监听相应的地址和端口。多个URL可以用来指定多个地址和端口来监听。etcd将从任何列出来的地址和端口上应答请求。

+ 默认: "http://localhost:2380"
+ 环境变量: ETCD_LISTEN_PEER_URLS
+ 例子: "http://10.0.0.1:2380"
+ 无效例子: "http://example.com:2380" (对于绑定域名无效)

### --listen-client-urls

用于监听客户端通讯的URL列表。这个标记告诉 etcd 在指定的 scheme://IP:port 上从接受其他节点的请求。scheme 可是 http 或者 https。如果IP被指定为0.0.0.0，etcd 将在所有接口上监听给定端口。如果指定IP地址和端口，etcd 将监听相应的地址和端口。多个URL可以用来指定多个地址和端口来监听。etcd将从任何列出来的地址和端口上应答请求。

+ 默认: "http://localhost:2379"
+ 环境变量: ETCD_LISTEN_CLIENT_URLS
+ 例子: "http://10.0.0.1:2379"
+ 无效例子: "http://example.com:2379" (对于绑定域名无效)

### --max-snapshots
+ 保持的快照文件的最大数量 (0 表示不限制)
+ 默认: 5
+ 环境变量: ETCD_MAX_SNAPSHOTS
+ 对于 windows 用户默认不限制，而且推荐手工降低到5（或者某些安全值)。

### --max-wals
+ 保持的 wal 文件的最大数量 (0 表示不限制)
+ 默认: 5
+ 环境变量: ETCD_MAX_WALS
+ 对于 windows 用户默认不限制，而且推荐手工降低到5（或者某些安全值)。

### --cors
+ 逗号分割的 CORS origin 白名单(CORS, cross-origin resource sharing/跨 origin 资源共享).
+ 默认: none
+ 环境变量: ETCD_CORS

## 集群标记

以 `--initial` 为前缀的标记用于启动([static bootstrap](clustering.md#静态), [discovery-service bootstrap](clustering.md#发现) 或 [runtime reconfiguration](runtime-configuration.md)) 新成员, 但是当重启已存在的成员是会被忽略。

`--discovery` 前缀标记在使用[发现服务](clustering.md#发现)时设置.

### --initial-advertise-peer-urls

通告给集群中其他节点的 URL 列表。这些地址用于集群节点间传递数据。至少要有一条地址其他节点可访问到。URL 中可以包含域名。

+ 默认: "http://localhost:2380"
+ 环境变量: ETCD_INITIAL_ADVERTISE_PEER_URLS
+ 例子: "http://example.com:2380, http://10.0.0.1:2380"

### --initial-cluster
+ 启动时初始化集群配置。
+ 默认: "default=http://localhost:2380"
+ 环境变量: ETCD_INITIAL_CLUSTER
+ key是 `--name` 标记的值。由于 `--name` 的默认值是 `default`, 所以key的默认值也为 `default`

### --initial-cluster-state

初始化集群状态("new" or "existing")。在初始化静态(initial static)或者 DNS 启动 (DNS bootstrapping) 期间为所有成员设置为 `new` 。如果这个选项被设置为 `existing` , etcd 将试图加入已有的集群。如果设置错误，etcd 将尝试启动但安全失败。

+ 默认: "new"
+ 环境变量: ETCD_INITIAL_CLUSTER_STATE

### --initial-cluster-token
+ 在启动期间用于 etcd 集群的初始化集群令牌(cluster token)。
+ 默认: "etcd-cluster"
+ 环境变量: ETCD_INITIAL_CLUSTER_TOKEN

### --advertise-client-urls
+ 用于通告给集群其他节点的 client URL。URL 中可以包含域名。
+ 默认: "http://localhost:2379"
+ 环境变量: ETCD_ADVERTISE_CLIENT_URLS
+ 例子: "http://example.com:2379, http://10.0.0.1:2379"

注意，如果来自集群成员的通告 URL 比如 http://localhost:2379，且正在使用 etcd 的 proxy 特性。这将导致循环，因为代理将转发请求给它自己直到它的资源(内存，文件描述符)最终耗尽。

### --discovery

用于启动集群的 discovery URL

+ 默认: none
+ 环境变量: ETCD_DISCOVERY

### --discovery-srv

用于启动集群的 DNS srv 域名。

+ 默认: none
+ 环境变量: ETCD_DISCOVERY_SRV

### --discovery-fallback

当发现服务失败时的期待行为("exit" 或 "proxy")。"proxy" 仅支持 v2 API。

+ 默认: "proxy"
+ 环境变量: ETCD_DISCOVERY_FALLBACK

### --discovery-proxy

用于请求到发现服务的 HTTP 代理。

+ 默认: none
+ 环境变量: ETCD_DISCOVERY_PROXY

### --strict-reconfig-check

拒绝将导致法定人数减少的重配置请求。

+ 默认: false
+ 环境变量: ETCD_STRICT_RECONFIG_CHECK

### --auto-compaction-retention

自动压缩用于 mvcc 键值存储的间隔，单位小时。 0 表示关闭自动压缩。
> **注：翻译不准确**

+ 默认: 0
+ 环境变量: ETCD_AUTO_COMPACTION_RETENTION

> 注： 对于服务注册等只保存运行时动态信息的场合，建议开启。完全没有理由损失存储空间和效率来保存之前的版本信息。推荐设置为1,每小时压缩一次。

## Proxy flags

以`--proxy` 为前缀的标记配置 etcd 以 [代理模式](https://github.com/coreos/etcd/blob/master/Documentation/v2/proxy.md) 运行。**"proxy" 仅支持 v2 API**。

### --proxy

代理模式设置("off", "readonly" or "on").

+ 默认: "off"
+ 环境变量: ETCD_PROXY

### --proxy-failure-wait

节点再被重新视为有效前，终端将被视为失败状态的时间(单位 毫秒)。

+ 默认: 5000
+ 环境变量: ETCD_PROXY_FAILURE_WAIT

### --proxy-refresh-interval

终端刷新间隔时间(单位 毫秒)

+ 默认: 30000
+ 环境变量: ETCD_PROXY_REFRESH_INTERVAL

### --proxy-dial-timeout

建立连接(dial)的超时时间(单位 毫秒)，0 表示禁用超时。

+ 默认: 1000
+ 环境变量 ETCD_PROXY_DIAL_TIMEOUT

### --proxy-write-timeout

写操作的超时时间(单位 毫秒)，0 表示禁用超时。

+ 默认: 5000
+ 环境变量: ETCD_PROXY_WRITE_TIMEOUT

### --proxy-read-timeout

读操作的超时时间(单位 毫秒)，0 表示禁用超时。

如果在使用 watch，请不要修改这个值，因为 watch 使用 long polling 进行请求。

+ 默认: 0
+ 环境变量: ETCD_PROXY_READ_TIMEOUT

## 安全标记

安全标记用于帮助 [搭建安全 etcd 集群](security.md).

### --ca-file [弃用]

客户端服务器 TLS CA文件的路径。`--ca-file ca.crt` 可以被 `--trusted-ca-file ca.crt --client-cert-auth` 替代。

+ 默认: none
+ 环境变量: ETCD_CA_FILE

### --cert-file

客户端服务器 TLS 证书文件的路径。

+ 默认: none
+ 环境变量: ETCD_CERT_FILE

### --key-file

客户端服务器 TLS key 文件的路径。

+ 默认: none
+ 环境变量: ETCD_KEY_FILE

### --client-cert-auth

开启客户端证书认证。

+ 默认: false
+ 环境变量: ETCD_CLIENT_CERT_AUTH

### --trusted-ca-file

客户端服务器 TLS 信任的 CA 文件路径。

+ 默认: none
+ 环境变量: ETCD_TRUSTED_CA_FILE

### --auto-tls

客户端 TLS 使用生成的证书。

+ 默认: false
+ 环境变量: ETCD_AUTO_TLS

### --peer-ca-file [弃用]

peer server TLS 证书文件的路径。 `--peer-ca-file ca.crt` 可以被 `--peer-trusted-ca-file ca.crt --peer-client-cert-auth` 替代。

+ 默认: none
+ 环境变量: ETCD_PEER_CA_FILE

### --peer-cert-file

peer server TLS 证书文件的路径。

+ 默认: none
+ 环境变量: ETCD_PEER_CERT_FILE

### --peer-key-file

peer server TLS key 文件的路径.

+ 默认: none
+ 环境变量: ETCD_PEER_KEY_FILE

### --peer-client-cert-auth

开启 peer client 证书验证.

+ 默认: false
+ 环境变量: ETCD_PEER_CLIENT_CERT_AUTH

### --peer-trusted-ca-file

peer server TLS 信任证书文件路径.

+ 默认: none
+ 环境变量: ETCD_PEER_TRUSTED_CA_FILE

### --peer-auto-tls

peer TLS使用生成的证书。

+ 默认: false
+ 环境变量: ETCD_PEER_AUTO_TLS

## 日志标记

### --debug

设置所有子包的默认日志级别为 DEBUG

+ 默认: false (所有包为 INFO)
+ 环境变量: ETCD_DEBUG

### --log-package-levels

设置特定的 etcd 子包的日志级别。例如 `etcdserver=WARNING,security=DEBUG`

+ 默认: none (所有包为 INFO)
+ 环境变量: ETCD_LOG_PACKAGE_LEVELS

## 不安全的标记

请谨慎使用不安全标记，因为可能破坏数据的一致性。
例如，它可能panic，如果集群中的其他成员还活着。
当使用这些标记时，请谨遵操作指南。

### --force-new-cluster

强制创建新的单一成员的集群。它提交配置修改来强制移除集群中的所有现有成员然后添加自身。此时需要设置 [restore a backup](https://github.com/coreos/etcd/blob/master/Documentation/v2/admin_guide.md#restoring-a-backup)。

+ 默认: false
+ 环境变量: ETCD_FORCE_NEW_CLUSTER

## 其他标记

### --version

打印版本并退出.

+ 默认: false

### --config-file

从文件中装载服务器配置.

+ 默认: none

## Profiling标记

### --enable-pprof

通过HTTP服务器开启profiling。地址是 client URL + "/debug/pprof/"

+ 默认: false

