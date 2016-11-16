# 在容器内运行etcd集群

> 注： 内容翻译自 [Run etcd clusters inside containers](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/container.md)

下列指南展示如何使用 [static bootstrap process](clustering.md#static) 来用rkt和docker运行 etcd 。

## rkt

### 运行单节点 etcd

下列 rkt 运行命令将在端口 2379 上暴露 etcd 客户端API，而在端口 2380上暴露伙伴API。

当配置 etcd 时使用 host IP地址。

```bash
export NODE1=192.168.1.21
```

信任 CoreOS [App Signing Key](https://coreos.com/security/app-signing-key/).

```bash
sudo rkt trust --prefix coreos.com/etcd
# gpg key fingerprint is: 18AD 5014 C99E F7E3 BA5F  6CE9 50BD D3E0 FC8A 365E
```

运行 `v3.0.6` 版本的 etcd or 或者指定其他发布版本。

```bash
sudo rkt run --net=default:IP=${NODE1} coreos.com/etcd:v3.0.6 -- -name=node1 -advertise-client-urls=http://${NODE1}:2379 -initial-advertise-peer-urls=http://${NODE1}:2380 -listen-client-urls=http://0.0.0.0:2379 -listen-peer-urls=http://${NODE1}:2380 -initial-cluster=node1=http://${NODE1}:2380
```

列出集群成员。

```bash
etcdctl --endpoints=http://192.168.1.21:2379 member list
```

### 运行3节点集群

使用 rkt 本地搭建 3 节点的集群， 使用`-initial-cluster` 标记.

```bash
export NODE1=172.16.28.21
export NODE2=172.16.28.22
export NODE3=172.16.28.23
```

```bash
# node 1
sudo rkt run --net=default:IP=${NODE1} coreos.com/etcd:v3.0.6 -- -name=node1 -advertise-client-urls=http://${NODE1}:2379 -initial-advertise-peer-urls=http://${NODE1}:2380 -listen-client-urls=http://0.0.0.0:2379 -listen-peer-urls=http://${NODE1}:2380 -initial-cluster=node1=http://${NODE1}:2380,node2=http://${NODE2}:2380,node3=http://${NODE3}:2380

# node 2
sudo rkt run --net=default:IP=${NODE2} coreos.com/etcd:v3.0.6 -- -name=node2 -advertise-client-urls=http://${NODE2}:2379 -initial-advertise-peer-urls=http://${NODE2}:2380 -listen-client-urls=http://0.0.0.0:2379 -listen-peer-urls=http://${NODE2}:2380 -initial-cluster=node1=http://${NODE1}:2380,node2=http://${NODE2}:2380,node3=http://${NODE3}:2380

# node 3
sudo rkt run --net=default:IP=${NODE3} coreos.com/etcd:v3.0.6 -- -name=node3 -advertise-client-urls=http://${NODE3}:2379 -initial-advertise-peer-urls=http://${NODE3}:2380 -listen-client-urls=http://0.0.0.0:2379 -listen-peer-urls=http://${NODE3}:2380 -initial-cluster=node1=http://${NODE1}:2380,node2=http://${NODE2}:2380,node3=http://${NODE3}:2380
```

检验集群健康并可以到达。

```bash
ETCDCTL_API=3 etcdctl --endpoints=http://172.16.28.21:2379,http://172.16.28.22:2379,http://172.16.28.23:2379 endpoint-health
```

### DNS

通过被本地解析器已知的 DNS 名称指向伙伴的产品集群必须挂载 [主机的 DNS 配置](https://coreos.com/kubernetes/docs/latest/kubelet-wrapper.html#customizing-rkt-options).

## Docker

为了暴露 etcd API 到 docker host 之外的客户端， 使用容器的 host IP 地址。请见[`docker inspect`](https://docs.docker.com/engine/reference/commandline/inspect) 来获取关于如何得到IP地址的更多细节. 或者, 为 `docker run` 命令指定 `--net=host` 标记来跳过放置容器在分隔的网络栈中。

```bash
# For each machine
ETCD_VERSION=v3.0.0
TOKEN=my-etcd-token
CLUSTER_STATE=new
NAME_1=etcd-node-0
NAME_2=etcd-node-1
NAME_3=etcd-node-2
HOST_1=10.20.30.1
HOST_2=10.20.30.2
HOST_3=10.20.30.3
CLUSTER=${NAME_1}=http://${HOST_1}:2380,${NAME_2}=http://${HOST_2}:2380,${NAME_3}=http://${HOST_3}:2380

# For node 1
THIS_NAME=${NAME_1}
THIS_IP=${HOST_1}
sudo docker run --net=host --name etcd quay.io/coreos/etcd:${ETCD_VERSION} \
	/usr/local/bin/etcd \
    --data-dir=data.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

# For node 2
THIS_NAME=${NAME_2}
THIS_IP=${HOST_2}
sudo docker run --net=host --name etcd quay.io/coreos/etcd:${ETCD_VERSION} \
	/usr/local/bin/etcd \
    --data-dir=data.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}

# For node 3
THIS_NAME=${NAME_3}
THIS_IP=${HOST_3}
sudo docker run --net=host --name etcd quay.io/coreos/etcd:${ETCD_VERSION} \
	/usr/local/bin/etcd \
    --data-dir=data.etcd --name ${THIS_NAME} \
	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
	--initial-cluster ${CLUSTER} \
	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
```

为了使用 API 版本3 来运行 `etcdctl` :

```bash
docker exec etcd /bin/sh -c "export ETCDCTL_API=3 && /usr/local/bin/etcdctl put foo bar"
```

## 裸机

为了在裸机上提供 3 节点 etcd 集群，可以在 [baremetal repo](https://github.com/coreos/coreos-baremetal/tree/master/examples) 中搜索有用的案例。

