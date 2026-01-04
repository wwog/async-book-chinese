# 应用：构建一个执行器

Rust 的 `Future` 是惰性的：除非被主动驱动至完成，否则它们不会做任何事情。驱动 future 完成的一种方法是在 `async` 函数内部 `.await` 它，但这只是将问题向上推了一层：谁来运行从顶级 `async` 函数返回的 future？答案是我们需要一个 `Future` 执行器。

`Future` 执行器获取一组顶级 `Future`，并在 `Future` 可以取得进展时通过调用 `poll` 将它们运行至完成。通常，执行器会先 `poll` 一次 future 以开始。当 `Future` 通过调用 `wake()` 指示它们准备好取得进展时，它们会被放回队列中，并再次调用 `poll`，重复此过程直到 `Future` 完成。

在本节中，我们将编写我们自己的简单执行器，能够并发运行大量顶级 future 至完成。

对于这个例子，我们依赖 `futures` crate 中的 `ArcWake` trait，它提供了一种构建 `Waker` 的简单方法。编辑 `Cargo.toml` 以添加新的依赖项：

```toml
[package]
name = "timer_future"
version = "0.1.0"
authors = ["XYZ Author"]
edition = "2021"

[dependencies]
futures = "0.3"
```

接下来，我们需要在 `src/main.rs` 顶部添加以下导入：

```rust,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:imports}}
```

我们的执行器将通过发送任务到通道来工作。执行器将从通道中提取事件并运行它们。当任务准备好做更多工作（被唤醒）时，它可以通过将自己放回通道来安排自己再次被轮询。

在这个设计中，执行器本身只需要任务通道的接收端。用户将获得一个发送端，以便他们可以生成新的 future。
任务本身只是可以重新调度自己的 future，所以我们将它们存储为与发送者配对的 future，任务可以使用该发送者将自己重新排队。

```rust,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:executor_decl}}
```

让我们也向 spawner 添加一个方法，以便轻松生成新的 future。
此方法将获取一个 future 类型，将其装箱，并在其中创建一个新的 `Arc<Task>`，该 `Arc<Task>` 可以排队到执行器上。

```rust,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:spawn_fn}}
```

要轮询 future，我们需要创建一个 `Waker`。
正如在 [任务唤醒章节] 中讨论的那样，`Waker` 负责在调用 `wake` 后安排任务再次被轮询。请记住，`Waker` 准确地告诉执行器哪个任务已准备就绪，允许它们仅轮询准备好取得进展的 future。创建新 `Waker` 的最简单方法是实现 `ArcWake` trait，然后使用 `waker_ref` 或 `.into_waker()` 函数将 `Arc<impl ArcWake>` 转换为 `Waker`。让我们为我们的任务实现 `ArcWake`，以允许它们被转换为 `Waker` 并被唤醒：

```rust,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:arcwake_for_task}}
```

当从 `Arc<Task>` 创建 `Waker` 时，对其调用 `wake()` 将导致 `Arc` 的副本被发送到任务通道上。然后我们的执行器需要获取任务并轮询它。让我们实现这一点：

```rust,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:executor_run}}
```

恭喜！我们现在有了一个工作的 future 执行器。我们甚至可以使用它来运行 `async/.await` 代码和自定义 future，例如我们之前编写的 `TimerFuture`：

```rust,edition2018,ignore
{{#include ../../examples/02_04_executor/src/lib.rs:main}}
```

[任务唤醒章节]: ./03_wakeups.md
