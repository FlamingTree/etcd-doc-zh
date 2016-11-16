# Watch service

Watch service提供观察键值对变化的支持。

在 rpc.proto 文件中 Watch service 定义如下：

```java
service Watch {
   // Watch 观察将要发生或者已经发生的事件。
   // 输入和输出都是流;输入流用于创建和取消观察，而输出流发送事件。
   // 一个观察 RPC 可以在一次性在多个key范围上观察，并为多个观察流化事件。
   // 整个事件历史可以从最后压缩修订版本开始观察。
  rpc Watch(stream WatchRequest) returns (stream WatchResponse) {}
}
```

只有一个Watch方法。



