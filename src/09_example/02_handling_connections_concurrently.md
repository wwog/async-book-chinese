# 并发处理连接
到目前为止，我们代码的问题在于 `listener.incoming()` 是一个阻塞迭代器。
当 `listener` 等待传入连接时，执行器无法运行其他 future，而且在处理完前一个连接之前，我们无法处理新连接。

为了解决这个问题，我们将把 `listener.incoming()` 从阻塞迭代器转换为非阻塞流。流类似于迭代器，但可以异步使用。
有关更多信息，请参阅 [关于流的章节](../05_streams/01_chapter.md)。

让我们用非阻塞的 `async_std::net::TcpListener` 替换阻塞的 `std::net::TcpListener`，并更新我们的连接处理程序以接受 `async_std::net::TcpStream`：
```rust,ignore
{{#include ../../examples/09_04_concurrent_tcp_server/src/main.rs:handle_connection}}
```

`TcpListener` 的异步版本为 `listener.incoming()` 实现了 `Stream` trait，这一变化带来了两个好处。
首先，`listener.incoming()` 不再阻塞执行器。
当没有传入的 TCP 连接需要处理时，执行器现在可以让出给其他挂起的 future。

第二个好处是，可以使用流的 `for_each_concurrent` 方法选择性地并发处理流中的元素。
在这里，我们将利用此方法并发处理每个传入请求。
我们需要从 `futures` crate 导入 `Stream` trait，所以我们的 Cargo.toml 现在看起来像这样：
```diff
+[dependencies]
+futures = "0.3"

 [dependencies.async-std]
 version = "1.6"
 features = ["attributes"]
```

现在，我们可以通过闭包函数传入 `handle_connection` 来并发处理每个连接。
闭包函数获取每个 `TcpStream` 的所有权，并在新的 `TcpStream` 可用时立即运行。
只要 `handle_connection` 不阻塞，慢速请求将不再阻止其他请求完成。
```rust,ignore
{{#include ../../examples/09_04_concurrent_tcp_server/src/main.rs:main_func}}
```
# 并行服务请求
到目前为止，我们的示例主要展示了协作式多任务并发（使用异步代码）作为抢占式多任务（使用线程）的替代方案。
然而，异步代码和线程并不是互斥的。
在我们的示例中，`for_each_concurrent` 并发处理每个连接，但在同一个线程上。
`async-std` crate 允许我们将任务生成到单独的线程上。
因为 `handle_connection` 既是 `Send` 又是无阻塞的，所以可以安全地与 `async_std::task::spawn` 一起使用。
它是这样的：
```rust
{{#include ../../examples/09_05_final_tcp_server/src/main.rs:main_func}}
```
现在我们同时使用协作式多任务并发和抢占式多任务来同时处理多个请求！
有关更多信息，请参阅 [关于多线程执行器的部分](../08_ecosystem/00_chapter.md#single-threading-vs-multithreading)。
