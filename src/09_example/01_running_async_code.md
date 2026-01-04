# 运行异步代码
HTTP 服务器应该能够并发地为多个客户端提供服务；也就是说，它不应该在处理当前请求之前等待先前的请求完成。
Rust 书籍通过创建一个线程池来 [解决这个问题](https://doc.rust-lang.org/book/ch20-02-multithreaded.html#turning-our-single-threaded-server-into-a-multithreaded-server)，其中每个连接都在其自己的线程上处理。
在这里，我们将使用异步代码来实现相同的效果，而不是通过添加线程来提高吞吐量。

让我们修改 `handle_connection`，通过将其声明为 `async fn` 来返回一个 future：
```rust,ignore
{{#include ../../examples/09_02_async_tcp_server/src/main.rs:handle_connection_async}}
```

将 `async` 添加到函数声明中会将其返回类型从单元类型 `()` 更改为实现 `Future<Output=()>` 的类型。

如果我们尝试编译它，编译器会警告我们它将不起作用：
```console
$ cargo check
    Checking async-rust v0.1.0 (file:///projects/async-rust)
warning: unused implementer of `std::future::Future` that must be used
  --> src/main.rs:12:9
   |
12 |         handle_connection(stream);
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: `#[warn(unused_must_use)]` on by default
   = note: futures do nothing unless you `.await` or poll them
```

因为我们没有 `await` 或 `poll` `handle_connection` 的结果，所以它永远不会运行。如果你运行服务器并在浏览器中访问 `127.0.0.1:7878`，你会看到连接被拒绝；我们的服务器没有处理请求。

我们不能在同步代码本身中 `await` 或 `poll` future。
我们需要一个异步运行时来处理调度和运行 future 至完成。
请查阅 [关于选择运行时的部分](../08_ecosystem/00_chapter.md) 以获取有关异步运行时、执行器和反应器的更多信息。
列出的任何运行时都适用于此项目，但对于这些示例，我们选择使用 `async-std` crate。

## 添加异步运行时
以下示例将演示重构同步代码以使用异步运行时；这里是 `async-std`。
来自 `async-std` 的 `#[async_std::main]` 属性允许我们编写异步 main 函数。
要使用它，请在 `Cargo.toml` 中启用 `async-std` 的 `attributes` 特性：
```toml
[dependencies.async-std]
version = "1.6"
features = ["attributes"]
```

作为第一步，我们将切换到异步 main 函数，并 `await` 异步版本的 `handle_connection` 返回的 future。
然后，我们将测试服务器的响应。
它是这样的：
```rust
{{#include ../../examples/09_02_async_tcp_server/src/main.rs:main_func}}
```
现在，让我们测试一下我们的服务器是否可以并发处理连接。
仅仅使 `handle_connection` 异步并不意味着服务器可以同时处理多个连接，我们很快就会明白为什么。

为了说明这一点，让我们模拟一个慢速请求。
当客户端向 `127.0.0.1:7878/sleep` 发出请求时，我们的服务器将休眠 5 秒：

```rust,ignore
{{#include ../../examples/09_03_slow_request/src/main.rs:handle_connection}}
```
这与书中的 [模拟慢速请求](https://doc.rust-lang.org/book/ch20-02-multithreaded.html#simulating-a-slow-request-in-the-current-server-implementation) 非常相似，但有一个重要的区别：
我们使用的是非阻塞函数 `async_std::task::sleep` 而不是阻塞函数 `std::thread::sleep`。
重要的是要记住，即使一段代码在 `async fn` 中运行并被 `await`，它仍然可能会阻塞。
为了测试我们的服务器是否并发处理连接，我们需要确保 `handle_connection` 是非阻塞的。

如果你运行服务器，你会看到对 `127.0.0.1:7878/sleep` 的请求将阻塞任何其他传入请求 5 秒！
这是因为在我们 `await` `handle_connection` 的结果时，没有其他并发任务可以取得进展。
在下一节中，我们将看到如何使用异步代码并发处理连接。
