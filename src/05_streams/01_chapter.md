# `Stream` Trait

`Stream` trait 类似于 `Future`，但在完成之前可以产生多个值，类似于标准库中的 `Iterator` trait：

```rust,ignore
{{#include ../../examples/05_01_streams/src/lib.rs:stream_trait}}
```

`Stream` 的一个常见例子是 `futures` crate 中通道类型的 `Receiver`。每次从 `Sender` 端发送值时，它都会产生 `Some(val)`，一旦 `Sender` 被丢弃并且所有待处理的消息都已接收，它将产生 `None`：

```rust,edition2018,ignore
{{#include ../../examples/05_01_streams/src/lib.rs:channels}}
```
