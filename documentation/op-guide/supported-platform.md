# 支持平台

> 注： 内容翻译自 [Supported platforms](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/supported-platform.md)

## 当前支持

下面的表单列出了对常见架构和系统的支持情况：

| 架构 | 操作系统 | 状态       | 维护者      |
| ------------ | ---------------- | ------------ | ---------------- |
| amd64        | Darwin           | 实现性 | etcd maintainers |
| amd64        | Linux            | 稳定       | etcd maintainers |
| amd64        | Windows          | 实现性 |                  |
| arm64        | Linux            | 实现性 | @glevand         |
| arm          | Linux            | 不稳定     |                  |
| 386          | Linux            | 不稳定     |                  |

* etcd-维护者列表 https://github.com/coreos/etcd/blob/master/MAINTAINERS.

试验性的平台几乎可以在实际中使用，并在 etcd 中存在一些平台相关的代码，但是没有完全遵守稳定支持政策( stable support policy)。不稳定平台有简单测试，但是少于实验性平台。为列出的架构和操作系统当前不支持，请当心！

### 支持新平台

对于 etcd 官方支持新的稳定平台，有一些要求是必须的，以保证可接受的质量：

1. 一个这个平台的 "官方" 维护者，有清晰的动机；必须有人负责照看这个平台。
2. 搭建构建的CI；etcd 必须编译通过
3. 搭建用于运行单元测试的CI；etcd 必须通过简单的测试。
4. 搭建用于运行集成测试的CI (TravisCI, SemaphoreCI 或 Jenkins)；etcd必须通过大量的测试。
5. (可选) 搭建功能测试集群；etcd 集群应该能通过压力测试。

### 32位和其他未支持系统

由于go runtime 的 bug，etcd 在32位系统存在一些已知的问题。阅读 [Go issue](https://github.com/golang/go/issues/599) 和 [atomic package](https://golang.org/pkg/sync/atomic/#pkg-note-BUG) 来获取详细信息。

为了避免不经意的运行可能不稳定的 etcd 服务器，在不稳定或者未支持架构上 `etcd` 将打印警告信息并立即退出，如果环境变量 `ETCD_UNSUPPORTED_ARCH` 没有设置为目标架构。

当前仅有 amd64 架构被 `etcd` 官方支持。


