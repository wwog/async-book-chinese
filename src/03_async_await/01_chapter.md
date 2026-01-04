# `async`/`.await`

在 [第一章] 中，我们简要介绍了 `async`/`.await`。
本章将更详细地讨论 `async`/`.await`，解释它是如何工作的，以及 `async` 代码与传统 Rust 程序有何不同。

`async`/`.await` 是 Rust 语法的特殊部分，它使得让出当前线程的控制权而不是阻塞成为可能，允许其他代码在等待操作完成时取得进展。

使用 `async` 主要有两种方式：`async fn` 和 `async` 块。
每个都返回一个实现 `Future` trait 的值：

```rust,edition2018,ignore
{{#include ../../examples/03_01_async_await/src/lib.rs:async_fn_and_block_examples}}
```

正如我们在第一章中看到的，`async` 主体和其他 future 是惰性的：它们在运行之前什么也不做。运行 `Future` 最常见的方法是 `.await` 它。当对 `Future` 调用 `.await` 时，它将尝试运行至完成。如果 `Future` 被阻塞，它将让出当前线程的控制权。当可以取得更多进展时，`Future` 将被执行器拾取并恢复运行，允许 `.await` 解析。

## `async` 生命周期

与传统函数不同，接受引用或其他非 `'static` 参数的 `async fn` 返回一个受参数生命周期限制的 `Future`：

```rust,edition2018,ignore
{{#include ../../examples/03_01_async_await/src/lib.rs:lifetimes_expanded}}
```

这意味着从 `async fn` 返回的 future 必须在其非 `'static` 参数仍然有效时被 `.await`。在调用函数后立即 `.await` future 的常见情况下（如 `foo(&x).await`），这不是问题。然而，如果存储 future 或将其发送到另一个任务或线程，这可能是一个问题。

将带有引用作为参数的 `async fn` 转换为 `'static` future 的一种常见解决方法是将参数与 `async fn` 的调用捆绑在 `async` 块内：

```rust,edition2018,ignore
{{#include ../../examples/03_01_async_await/src/lib.rs:static_future_with_borrow}}
```

通过将参数移动到 `async` 块中，我们延长了它的生命周期以匹配从调用 `good` 返回的 `Future` 的生命周期。

## `async move`

`async` 块和闭包允许使用 `move` 关键字，就像普通闭包一样。`async move` 块将获取其引用的变量的所有权，允许它比当前作用域存活得更久，但放弃了与其他代码共享这些变量的能力：

```rust,edition2018,ignore
{{#include ../../examples/03_01_async_await/src/lib.rs:async_move_examples}}
```

## 在多线程执行器上 `.await`

请注意，当使用多线程 `Future` 执行器时，`Future` 可能会在线程之间移动，因此 `async` 主体中使用的任何变量必须能够在线程之间传输，因为任何 `.await` 都可能导致切换到新线程。

这意味着使用 `Rc`、`&RefCell` 或任何其他未实现 `Send` trait 的类型是不安全的，包括对未实现 `Sync` trait 的类型的引用。

（警告：只要这些类型在调用 `.await` 期间不在作用域内，就可以使用它们。）

同样，在 `.await` 期间持有传统的非 future 感知锁并不是一个好主意，因为它可能导致线程池锁定：一个任务可能获取锁，`.await` 并让出给执行器，允许另一个任务尝试获取锁并导致死锁。为了避免这种情况，请使用 `futures::lock` 中的 `Mutex` 而不是 `std::sync` 中的那个。

[第一章]: ../01_getting_started/04_async_await_primer.md
