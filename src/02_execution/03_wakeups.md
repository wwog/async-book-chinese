# 使用 `Waker` 唤醒任务

通常情况下，future 无法在第一次被 `poll` 时完成。当这种情况发生时，future 需要确保一旦准备好取得更多进展，它就会再次被轮询。这是通过 `Waker` 类型完成的。

每次轮询 future 时，它都是作为 "任务" 的一部分进行轮询的。任务是已提交给执行器的顶级 future。

`Waker` 提供了一个 `wake()` 方法，可用于告诉执行器应该唤醒关联的任务。当调用 `wake()` 时，执行器知道与 `Waker` 关联的任务准备好取得进展，并且应该再次轮询其 future。

`Waker` 还实现了 `clone()`，以便可以复制和存储它。

让我们尝试使用 `Waker` 实现一个简单的计时器 future。

## 应用：构建一个计时器

为了举例说明，我们将在创建计时器时启动一个新线程，休眠所需的时间，然后在时间窗口过去后向计时器 future 发出信号。

首先，使用 `cargo new --lib timer_future` 启动一个新项目，并将我们需要开始的导入添加到 `src/lib.rs`：

```rust
{{#include ../../examples/02_03_timer/src/lib.rs:imports}}
```

让我们从定义 future 类型本身开始。我们的 future 需要一种方法让线程传达计时器已过且 future 应该完成的信息。
我们将使用共享的 `Arc<Mutex<..>>` 值在线程和 future 之间进行通信。

```rust,ignore
{{#include ../../examples/02_03_timer/src/lib.rs:timer_decl}}
```

现在，让我们实际编写 `Future` 实现！

```rust,ignore
{{#include ../../examples/02_03_timer/src/lib.rs:future_for_timer}}
```

很简单，对吧？如果线程设置了 `shared_state.completed = true`，我们就完成了！否则，我们克隆当前任务的 `Waker` 并将其传递给 `shared_state.waker`，以便线程可以唤醒任务。

重要的是，我们必须在每次轮询 future 时更新 `Waker`，因为 future 可能已经移动到具有不同 `Waker` 的不同任务中。当 future 在被轮询后在任务之间传递时，就会发生这种情况。

最后，我们需要 API 来实际构造计时器并启动线程：

```rust,ignore
{{#include ../../examples/02_03_timer/src/lib.rs:timer_new}}
```

哇！这就是构建一个简单的计时器 future 所需的一切。现在，要是我们有一个执行器来运行 future 就好了...
