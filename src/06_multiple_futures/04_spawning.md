# `Spawning`

Spawning 允许你在后台运行一个新的异步任务。这允许我们在它运行时继续执行其他代码。

假设我们有一个 Web 服务器，想要在不阻塞主线程的情况下接受连接。
为了实现这一点，我们可以使用 `async_std::task::spawn` 函数来创建并运行一个处理连接的新任务。此函数接受一个 future 并返回一个 `JoinHandle`，可用于在任务完成后等待其结果。

```rust,edition2018
{{#include ../../examples/06_04_spawning/src/lib.rs:example}}
```

`spawn` 返回的 `JoinHandle` 实现了 `Future` trait，因此我们可以 `.await` 它以获取任务的结果。
这将阻塞当前任务，直到生成的任务完成。如果未等待任务，你的程序将继续执行而不等待任务，如果函数在任务完成之前完成，则取消该任务。

```rust,edition2018
{{#include ../../examples/06_04_spawning/src/lib.rs:join_all}}
```

要在主任务和生成的任务之间进行通信，我们可以使用所使用的异步运行时提供的通道。
