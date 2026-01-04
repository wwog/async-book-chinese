# `async`/`.await` 入门

`async`/`.await` 是 Rust 的内置工具，用于编写看起来像同步代码的异步函数。`async` 将代码块转换为实现名为 `Future` 的 trait 的状态机。在同步方法中调用阻塞函数会阻塞整个线程，而阻塞的 `Future` 将让出线程的控制权，允许其他 `Future` 运行。

让我们向 `Cargo.toml` 文件添加一些依赖项：

```toml
{{#include ../../examples/01_04_async_await_primer/Cargo.toml:9:10}}
```

要创建异步函数，可以使用 `async fn` 语法：

```rust,edition2018
async fn do_something() { /* ... */ }
```

`async fn` 返回的值是一个 `Future`。为了让任何事情发生，`Future` 需要在执行器上运行。

```rust,edition2018
{{#include ../../examples/01_04_async_await_primer/src/lib.rs:hello_world}}
```

在 `async fn` 内部，你可以使用 `.await` 等待另一个实现 `Future` trait 的类型完成，例如另一个 `async fn` 的输出。与 `block_on` 不同，`.await` 不会阻塞当前线程，而是异步等待 future 完成，如果 future 当前无法取得进展，则允许其他任务运行。

例如，假设我们有三个 `async fn`：`learn_song`、`sing_song` 和 `dance`：

```rust,ignore
async fn learn_song() -> Song { /* ... */ }
async fn sing_song(song: Song) { /* ... */ }
async fn dance() { /* ... */ }
```

学习、唱歌和跳舞的一种方法是分别阻塞每一个：

```rust,ignore
{{#include ../../examples/01_04_async_await_primer/src/lib.rs:block_on_each}}
```

然而，这种方式并没有提供最佳性能——我们一次只做一件事！显然，我们必须先学会这首歌才能唱，但可以在学习和唱歌的同时跳舞。为此，我们可以创建两个可以并发运行的独立 `async fn`：

```rust,ignore
{{#include ../../examples/01_04_async_await_primer/src/lib.rs:block_on_main}}
```

在这个例子中，学习歌曲必须发生在唱歌之前，但学习和唱歌都可以与跳舞同时发生。如果在 `learn_and_sing` 中使用 `block_on(learn_song())` 而不是 `learn_song().await`，那么在 `learn_song` 运行时，线程将无法做任何其他事情。
这将使得同时跳舞变得不可能。通过 `.await` `learn_song` future，如果 `learn_song` 被阻塞，我们允许其他任务接管当前线程。这使得在同一个线程上并发运行多个 future 直至完成成为可能。
