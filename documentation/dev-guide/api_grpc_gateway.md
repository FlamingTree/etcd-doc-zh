# gRPC网关

> 注： 内容翻译自 [gRPC gateway](https://github.com/coreos/etcd/blob/master/Documentation/dev-guide/api_grpc_gateway.md)

## 为什么用 grpc-gateway

etcd v3 使用 [gRPC](http://www.grpc.io/) 作为它的消息协议。etcd 项目包括基于 gRPC 的 [Go client](https://github.com/coreos/etcd/tree/master/clientv3) 和 命令行工具 etcdctl，通过 gRPC 和 etcd 集群通讯。对于不支持 gRPC 支持的语言，etcd 提供 JSON 的 [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)。这个网关提供 RESTful 代理，翻译 HTTP/JSON 请求为 gRPC 消息。


## 使用 grpc-gateway

网关接受 etcd 的 [protocol buffer](api_reference_v3.md) 的 [JSON mapping](https://developers.google.com/protocol-buffers/docs/proto3#json) 。注意  `key` 和 `value` 字段被定义为 byte 数组，因此必须在 JSON 中以 base64 编码.

```bash
< < COMMENT
https://www.base64encode.org/
foo is 'Zm9v' in Base64
bar is 'YmFy'
COMMENT

curl -L http://localhost:2379/v3alpha/kv/put \
	-X POST -d '{"key": "Zm9v", "value": "YmFy"}'

curl -L http://localhost:2379/v3alpha/kv/range \
	-X POST -d '{"key": "Zm9v"}'
```


## Swagger

生成的 [Swagger](http://swagger.io/) API 定义可以在 [rpc.swagger.json](https://github.com/coreos/etcd/blob/master/Documentation/dev-guide/apispec/swagger/rpc.swagger.json) 找到.

