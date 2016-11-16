# API 参考文档

## 前言

这份文档来自 ectd3 的 [API reference](https://github.com/coreos/etcd/blob/master/Documentation/dev-guide/api_reference_v3.md)。

原文内容是从 .proto 文件生成的，所有的service和他们的方法定义/message定义混合在一起构成一个巨大的单页文件。

看起来非常累......

所以干脆拆分出来，按照服务/方法分开，看起来方便。

主要内容在 message 的字段定义上，我翻译时没有复制原文中从 .proto 文件生成的表格，而是直接在 .proto 文件的 message 定义上翻译，感觉更直观一些。

> 注： 我假定阅读这份APi的同学对 proto3 的基本语法有初步的了解，如果没有，可以参考我的另外一份 [proto3学习笔记](https://skyao.gitbooks.io/leaning-proto3/)。

## 内容

已经整理并翻译的内容：

* [KV service](kv/kv_service.md)
* [Watch service](watch/watch_service.md)
* [Lease service](lease/lease_service.md)

