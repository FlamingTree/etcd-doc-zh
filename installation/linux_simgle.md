# Linux单机部署

> 注： 参考 [etcd releases页面](https://github.com/coreos/etcd/releases/) 的说明

## 下载

执行下面的命令，下载(大概10M)解压即可，无需安装：

```bash
curl -L https://github.com/coreos/etcd/releases/download/v3.0.6/etcd-v3.0.6-linux-amd64.tar.gz -o etcd-v3.0.6-linux-amd64.tar.gz
tar xzvf etcd-v3.0.6-linux-amd64.tar.gz
mv etcd-v3.0.6-linux-amd64 etcd
cd etcd
./etcd --version

etcd Version: 3.0.6
Git SHA: 9efa00d
Go Version: go1.6.3
Go OS/Arch: linux/amd64
```

> 注： 以 etcd-v3.0.6 为例，后续更新版本时可能细节有所不同。

安装目录文件列表如下：

```bash
$ ls
default.etcd   etcd     README-etcdctl.md  READMEv2-etcdctl.md
Documentation  etcdctl  README.md
```

## 运行

直接运行命令 `./etcd` 就可以启动了，非常简单：

```bash
$ ./etcd
2016-08-29 15:25:10.987041 W | flags: unrecognized environment variable ETCDCTL_API=3
2016-08-29 15:25:10.987143 I | etcdmain: etcd Version: 3.0.6
2016-08-29 15:25:10.987160 I | etcdmain: Git SHA: 9efa00d
2016-08-29 15:25:10.987177 I | etcdmain: Go Version: go1.6.3
2016-08-29 15:25:10.987192 I | etcdmain: Go OS/Arch: linux/amd64
2016-08-29 15:25:10.987216 I | etcdmain: setting maximum number of CPUs to 8, total number of available CPUs is 8
2016-08-29 15:25:10.987232 W | etcdmain: no data-dir provided, using default data-dir ./default.etcd
2016-08-29 15:25:10.987588 I | etcdmain: listening for peers on http://localhost:2380
2016-08-29 15:25:10.987651 I | etcdmain: listening for client requests on localhost:2379
2016-08-29 15:25:10.990811 I | etcdserver: name = default
2016-08-29 15:25:10.990843 I | etcdserver: data dir = default.etcd
2016-08-29 15:25:10.990859 I | etcdserver: member dir = default.etcd/member
2016-08-29 15:25:10.990874 I | etcdserver: heartbeat = 100ms
2016-08-29 15:25:10.990888 I | etcdserver: election = 1000ms
2016-08-29 15:25:10.990904 I | etcdserver: snapshot count = 10000
2016-08-29 15:25:10.990934 I | etcdserver: advertise client URLs = http://localhost:2379
2016-08-29 15:25:10.990952 I | etcdserver: initial advertise peer URLs = http://localhost:2380
2016-08-29 15:25:10.990975 I | etcdserver: initial cluster = default=http://localhost:2380
2016-08-29 15:25:10.995843 I | etcdserver: starting member 8e9e05c52164694d in cluster cdf818194e3a8c32
2016-08-29 15:25:10.996018 I | raft: 8e9e05c52164694d became follower at term 0
2016-08-29 15:25:10.996238 I | raft: newRaft 8e9e05c52164694d [peers: [], term: 0, commit: 0, applied: 0, lastindex: 0, lastterm: 0]
2016-08-29 15:25:10.996271 I | raft: 8e9e05c52164694d became follower at term 1
2016-08-29 15:25:11.013314 I | etcdserver: starting server... [version: 3.0.6, cluster version: to_be_decided]
2016-08-29 15:25:11.014091 I | membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32
2016-08-29 15:25:11.197323 I | raft: 8e9e05c52164694d is starting a new election at term 1
2016-08-29 15:25:11.197377 I | raft: 8e9e05c52164694d became candidate at term 2
2016-08-29 15:25:11.197388 I | raft: 8e9e05c52164694d received vote from 8e9e05c52164694d at term 2
2016-08-29 15:25:11.197408 I | raft: 8e9e05c52164694d became leader at term 2
2016-08-29 15:25:11.197420 I | raft: raft.node: 8e9e05c52164694d elected leader 8e9e05c52164694d at term 2
2016-08-29 15:25:11.197820 I | etcdserver: published {Name:default ClientURLs:[http://localhost:2379]} to cluster cdf818194e3a8c32
2016-08-29 15:25:11.197845 I | etcdmain: ready to serve client requests
2016-08-29 15:25:11.197979 I | etcdserver: setting up the initial cluster version to 3.0
2016-08-29 15:25:11.198202 N | etcdmain: serving insecure client requests on localhost:2379, this is strongly discouraged!
2016-08-29 15:25:11.199390 N | membership: set the initial cluster version to 3.0
2016-08-29 15:25:11.199458 I | api: enabled capabilities for version 3.0
```

默认使用2379端口为客户端提供通讯， 并使用端口2380来进行服务器间通讯。

查看当前安装的版本：

```bash
$ ./etcd --version
etcd Version: 3.0.6
Git SHA: 9efa00d
Go Version: go1.6.3
Go OS/Arch: linux/amd64
```

## 配置

为了方便使用，将 etcd 加入 PATH，另外设置 ETCDCTL_API 为3(后面解释)。

在 `/etc/profile` 中加入以下内容：

```bash
# etcd
export PATH=/home/sky/work/soft/etcd:$PATH
export ETCDCTL_API=3
```

然后执行 `source /etc/profile` 重新加载。

## 客户端访问

### 配置etcdctl

etcdctl 是 etcd 的客户端命令行。

> 特别提醒：使用 etcdctl 前，**务必设置环境变量 `ETCDCTL_API=3` **!

注意：如果不设置 `ETCDCTL_API=3`，则默认是的API版本是2：

```bash
$ ./etcdctl --version
etcdctl version: 3.0.6
API version: 2
```

正确设置后，API版本变成3：

```bash
$ etcdctl version
etcdctl version: 3.0.6
API version: 3.0
```

### 使用etcdctl

通过下面的put和get命令来验证连接并操作etcd：

```bash
$ ./etcdctl put aaa 1
OK
$ ./etcdctl get aaa
aaa
1
```

## 总结

上面操作完成之后，就有一个可运行的简单 etcd 服务器和一个可用的 etcdctl 客户端。
