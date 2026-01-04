# `Future` Trait

`Future` trait 是 Rust 异步编程的核心。
`Future` 是一个可以产生值（尽管该值可能为空，例如 `()`）的异步计算。`Future` trait 的一个 *简化* 版本可能如下所示：

```rust
{{#include ../../examples/02_02_future_trait/src/lib.rs:simple_future}}
```

可以通过调用 `poll` 函数来推进 Future，该函数将尽可能地驱动 future 完成。如果 future 完成，它将返回 `Poll::Ready(result)`。如果 future 尚未完成，它将返回 `Poll::Pending` 并安排在 `Future` 准备好取得更多进展时调用 `wake()` 函数。当 `wake()` 被调用时，驱动 `Future` 的执行器将再次调用 `poll`，以便 `Future` 可以取得更多进展。

如果没有 `wake()`，执行器将无法知道特定的 future 何时可以取得进展，并且必须不断轮询每个 future。有了 `wake()`，执行器确切地知道哪些 future 准备好被 `poll`。

例如，考虑我们要从可能已有或没有数据可用的套接字读取的情况。如果有数据，我们可以读取它并返回 `Poll::Ready(data)`，但如果没有数据准备好，我们的 future 将被阻塞并且无法再取得进展。当没有数据可用时，我们必须注册 `wake` 以便在套接字上有数据准备好时被调用，这将告诉执行器我们的 future 准备好取得进展。
一个简单的 `SocketRead` future 可能如下所示：

```rust,ignore
{{#include ../../examples/02_02_future_trait/src/lib.rs:socket_read}}
```

这种 `Future` 模型允许组合多个异步操作而无需中间分配。一次运行多个 future 或将 future 链接在一起可以通过无分配状态机来实现，如下所示：

```rust,ignore
{{#include ../../examples/02_02_future_trait/src/lib.rs:join}}
```

这展示了如何在不需要单独分配的情况下同时运行多个 future，从而实现更高效的异步程序。
同样，多个顺序 future 可以一个接一个地运行，如下所示：

```rust,ignore
{{#include ../../examples/02_02_future_trait/src/lib.rs:and_then}}
```

这些示例展示了如何使用 `Future` trait 来表达异步控制流，而无需多个分配的对象和深度嵌套的回调。在介绍了基本的控制流之后，让我们来谈谈真正的 `Future` trait 以及它有何不同。

```rust,ignore
{{#include ../../examples/02_02_future_trait/src/lib.rs:real_future}}
```

你会注意到的第一个变化是我们的 `self` 类型不再是 `&mut Self`，而是变成了 `Pin<&mut Self>`。我们将在后面的章节中详细讨论 pinning，但现在只需知道它允许我们创建不可移动的 future。不可移动的对象可以在其字段之间存储指针，例如 `struct MyFut { a: i32, ptr_to_a: *const i32 }`。Pinning 是启用 async/await 所必需的。

其次，`wake: fn()` 变成了 `&mut Context<'_>`。在 `SimpleFuture` 中，我们使用函数指针 (`fn()`) 调用来告诉 future 执行器应该轮询有问题的 future。然而，由于 `fn()` 只是一个函数指针，它不能存储任何关于 *哪个* `Future` 调用了 `wake` 的数据。

在现实世界的场景中，像 Web 服务器这样的复杂应用程序可能有数千个不同的连接，它们的唤醒都应该单独管理。`Context` 类型通过提供对 `Waker` 类型值的访问来解决这个问题，该值可用于唤醒特定任务。
