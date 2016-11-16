# Mac单机部署

> 注： 参考 [etcd releases页面](https://github.com/coreos/etcd/releases/) 的说明

## 下载

执行下面的命令，下载解压即可，无需安装：

```bash
ETCD_VER=v3.0.15
DOWNLOAD_URL=https://github.com/coreos/etcd/releases/download
curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-darwin-amd64.zip -o /tmp/etcd-${ETCD_VER}-darwin-amd64.zip
mkdir -p /tmp/test-etcd && unzip /tmp/etcd-${ETCD_VER}-darwin-amd64.zip -d /tmp && mv /tmp/etcd-${ETCD_VER}-darwin-amd64/* /tmp/test-etcd

/tmp/test-etcd/etcd --version
```

> 注： 以 etcd-v3.0.15 为例，后续更新版本时可能细节有所不同。

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
2016-11-16 21:29:10.421838 I | etcdmain: etcd Version: 3.0.15
2016-11-16 21:29:10.421915 I | etcdmain: Git SHA: fc00305
2016-11-16 21:29:10.421921 I | etcdmain: Go Version: go1.6.3
2016-11-16 21:29:10.421926 I | etcdmain: Go OS/Arch: darwin/amd64
2016-11-16 21:29:10.421932 I | etcdmain: setting maximum number of CPUs to 8, total number of available CPUs is 8
2016-11-16 21:29:10.421940 W | etcdmain: no data-dir provided, using default data-dir ./default.etcd
2016-11-16 21:29:10.422007 N | etcdmain: the server is already initialized as member before, starting as etcd member...
2016-11-16 21:29:10.423738 I | etcdmain: listening for peers on http://localhost:2380
2016-11-16 21:29:10.424854 I | etcdmain: listening for client requests on localhost:2379
2016-11-16 21:29:10.425885 I | etcdserver: name = default
2016-11-16 21:29:10.425899 I | etcdserver: data dir = default.etcd
2016-11-16 21:29:10.425905 I | etcdserver: member dir = default.etcd/member
2016-11-16 21:29:10.425910 I | etcdserver: heartbeat = 100ms
2016-11-16 21:29:10.425914 I | etcdserver: election = 1000ms
2016-11-16 21:29:10.425919 I | etcdserver: snapshot count = 10000
2016-11-16 21:29:10.425929 I | etcdserver: advertise client URLs = http://localhost:2379
2016-11-16 21:29:10.447242 I | etcdserver: restarting member 8e9e05c52164694d in cluster cdf818194e3a8c32 at commit index 120
2016-11-16 21:29:10.447309 I | raft: 8e9e05c52164694d became follower at term 3
2016-11-16 21:29:10.447332 I | raft: newRaft 8e9e05c52164694d [peers: [], term: 3, commit: 120, applied: 0, lastindex: 120, lastterm: 3]
2016-11-16 21:29:10.448938 I | etcdserver: starting server... [version: 3.0.15, cluster version: to_be_decided]
2016-11-16 21:29:10.449039 E | etcdserver: cannot monitor file descriptor usage (cannot get FDUsage on darwin)
2016-11-16 21:29:10.449564 I | membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32
2016-11-16 21:29:10.449658 N | membership: set the initial cluster version to 3.0
2016-11-16 21:29:10.449693 I | api: enabled capabilities for version 3.0
2016-11-16 21:29:11.248291 I | raft: 8e9e05c52164694d is starting a new election at term 3
2016-11-16 21:29:11.248338 I | raft: 8e9e05c52164694d became candidate at term 4
2016-11-16 21:29:11.248350 I | raft: 8e9e05c52164694d received vote from 8e9e05c52164694d at term 4
2016-11-16 21:29:11.248367 I | raft: 8e9e05c52164694d became leader at term 4
2016-11-16 21:29:11.248380 I | raft: raft.node: 8e9e05c52164694d elected leader 8e9e05c52164694d at term 4
2016-11-16 21:29:11.261511 I | etcdserver: published {Name:default ClientURLs:[http://localhost:2379]} to cluster cdf818194e3a8c32
2016-11-16 21:29:11.261537 I | etcdmain: ready to serve client requests
2016-11-16 21:29:11.262789 N | etcdmain: serving insecure client requests on localhost:2379, this is strongly discouraged!
```

默认使用2379端口为客户端提供通讯， 并使用端口2380来进行服务器间通讯。

查看当前安装的版本：

```bash
$ ./etcd --version
etcd Version: 3.0.15
Git SHA: fc00305
Go Version: go1.6.3
Go OS/Arch: darwin/amd64
```

## 配置

为了方便使用，将 etcd 加入 PATH，另外设置 ETCDCTL_API 为3(后面解释)。

在 `/etc/profile` 中加入以下内容：

```bash
# etcd
export PATH=~/bi/etcd:$PATH
export ETCDCTL_API=3
```

然后执行 `source /etc/profile` 重新加载。

## 客户端访问

### 配置etcdctl

etcdctl 是 etcd 的客户端命令行。

> 特别提醒：使用 etcdctl 前，**务必设置环境变量 `ETCDCTL_API=3` **!

注意：如果不设置 `ETCDCTL_API=3`，则默认是的API版本是2：

```bash
$ ./etcdctl version
etcdctl version: 3.0.15
API version: 2.0
```

正确设置后，API版本变成3：

```bash
$ etcdctl version
etcdctl version: 3.0.15
API version: 3.0
```

### 使用etcdctl

通过下面的put和get命令来验证连接并操作etcd：

```bash
$ ./etcdctl put x 1
OK
$ ./etcdctl get x
aaa
1
```

## 总结

上面操作完成之后，就有一个可运行的简单 etcd 服务器和一个可用的 etcdctl 客户端。


