# 版本

> 注： 内容翻译自 [Versioning](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/versioning.md)

### 服务版本

etcd 使用 [semantic versioning](http://semver.org)。新的小版本可能增加额外功能到API。

使用 `etcdctl` 获取运行中的 etcd 集群的版本：

```bash
ETCDCTL_API=3 etcdctl --endpoints=127.0.0.1:2379 endpoint status
```

### API 版本

在 3.0.0 发布值偶 `v3` API 应答将不会更改， 但是随着时间的推移将加入新的功能。
