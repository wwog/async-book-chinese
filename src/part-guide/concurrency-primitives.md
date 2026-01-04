# 并发组合 futures

在本章中，我们将介绍更多组合 futures 的方法。特别是，一些可以并发执行 futures（但不是并行）的新方法。表面上看，我们在本章介绍的新函数/宏非常简单。然而，基本概念可能非常微妙。我们将从回顾 futures、并发和并行开始，但你可能还想重温前面比较[并发与并行](concurrency.md#concurrency-and-parallelism)的部分。

Future 是延迟计算。Future 可以通过使用 `await` 来推进，它将控制权移交给运行时，导致当前任务等待计算结果。如果 `a` 和 `b` 是 futures，那么它们可以通过先 `await` 一个再 `await` 另一个来顺序组合（即，组合成一个先执行 `a` 完成然后执行 `b` 完成的 future）：`async { a.await; b.await}`。

我们也看到了使用 `spawn` 并行组合 futures：`async { let a = spawn(a); let b = spawn(b); (a.await, b.await)}` 并行运行这两个 futures。请注意，元组中的 `await` 不是在等待 futures 本身，而是在等待 `JoinHandle` 以在 futures 完成时获取结果。

在本章中，我们介绍了两种在没有并行的情况下并发组合 futures 的方法：`join` 和 `select`/`race`。在这两种情况下，futures 通过时间片并发运行；每个组合的 futures 轮流执行，然后下一个获得机会。这是在*不涉及异步运行时*的情况下完成的（因此没有多个操作系统线程，也没有任何并行的可能性）。组合结构在本地交错 futures。你可以将这些结构视为微型执行器，它们在单个异步任务中执行其组件 futures。

join 和 select/race 之间的根本区别在于它们如何处理完成工作的 futures：join 在所有 futures 完成时完成，select/race 在一个 future 完成时完成（所有其他的都被取消）。两者都有处理错误的变体。

这些结构（或类似概念）通常与 streams 一起使用，我们将在下面简要提及，但我们将在 [streams 章节](streams.md)中详细讨论。

如果你想要并行（或者你没有明确不想要并行），派生任务通常是这些组合结构的更简单替代方案。派生任务通常更不容易出错，更通用，性能更可预测。另一方面，派生本质上不那么[结构化](../part-reference/structured.md)，这可能使生命周期和资源管理更难推理。

值得更深入地考虑性能问题。并发组合的潜在性能问题是时间共享的公平性。如果你的程序中有 100 个任务，那么通常共享资源的最佳方式是每个任务获得 1% 的处理器时间（或者如果任务都在等待，那么每个任务都有相同的被唤醒机会）。如果你派生 100 个任务，那么通常会发生这种情况（大致如此）。但是，如果你派生两个任务并在其中一个任务上 join 99 个 futures，那么调度程序将只知道两个任务，一个任务将获得 50% 的时间，而 99 个 futures 将各获得 0.5%。

通常任务的分布不会如此偏颇，而且我们经常将 join/select/etc. 用于超时之类的事情，这种行为实际上是可取的。但这值得考虑，以确保你的程序具有你想要的性能特征。


## Join

Tokio 的 [`join` 宏](https://docs.rs/tokio/latest/tokio/macro.join.html) 接受一个 futures 列表并并发运行它们直到完成（将所有结果作为元组返回）。当所有 futures 完成时它返回。Futures 总是在同一个线程上执行（并发而不是并行）。

这是一个简单的例子：

```rust,norun
async fn main() {
  let (result_1, result_2) = join!(do_a_thing(), do_a_thing());
  // 使用 `result_1` 和 `result_2`。
}
```

在这里，`do_a_thing` 的两次执行并发发生，结果在它们都完成时准备好。注意我们没有 `await` 来获取结果。`join!` 隐式地 await 它的 futures 并产生一个值。它不创建一个 future。你仍然需要在异步上下文中使用它（例如，在异步函数内部）。

虽然你在上面的例子中看不到，但 `join!` 接受求值为 futures 的表达式[^into]。`join` 不会在其主体中创建异步上下文，你不应该 `await` 传递给 `join` 的 futures（否则它们将在 joined futures 之前被求值）。

因为所有 futures 都在同一个线程上执行，如果任何 future 阻塞了线程，那么它们都无法取得进展。如果使用互斥锁或其他锁，如果一个 future 正在等待另一个 future 持有的锁，这很容易导致死锁。

[`join`](https://docs.rs/tokio/latest/tokio/macro.join.html) 不关心 futures 的结果。特别是，如果一个 future 被取消或返回错误，它不会影响其他的——它们继续执行。如果你想要“快速失败”行为，请使用 [`try_join`](https://docs.rs/tokio/latest/tokio/macro.try_join.html)。`try_join` 的工作方式类似于 `join`，但是，如果任何 future 返回 `Err`，那么所有其他 futures 都将被取消，并且 `try_join` 立即返回错误。

在前面关于 [async/await](async-await.md) 的章节中，我们使用“join”这个词来讨论 join 派生的任务。顾名思义，joining futures 和任务是相关的：joining 意味着我们并发执行多个 futures 并在继续之前等待结果。语法不同：使用 `JoinHandle` 与 `join` 宏，但想法是相似的。关键区别在于，当 joining 任务时，任务并发且并行执行，而使用 `join!` 时，futures 并发执行但不并行执行。此外，派生的任务在运行时的调度程序上调度，而使用 `join!` 时，futures 在本地“调度”（在同一个任务上并在宏执行的时间范围内）。另一个区别是，如果派生的任务恐慌，恐慌会被运行时捕获，但如果 `join` 中的 future 恐慌，那么整个任务都会恐慌。


### 替代方案

并发运行 futures 并收集它们的结果是一个常见需求。你应该使用 `spawn` 和 `JoinHandle`，除非你有充分的理由不这样做（即，你明确不想要并行，即使那样你也可能更喜欢使用 [`spawn_local`](https://docs.rs/tokio/latest/tokio/task/fn.spawn_local.html)）。[`JoinSet`](https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html) 抽象以类似于 `join!` 的方式管理此类派生任务。

大多数运行时（和 [futures.rs](https://docs.rs/futures/latest/futures/macro.join.html)）都有相当于 Tokio 的 `join` 宏，它们的行为大多相同。还有 `join` 函数，类似于宏，但不那么灵活。例如，futures.rs 有用于 join 两个 futures 的 [`join`](https://docs.rs/futures/latest/futures/future/fn.join.html)，用于 join 明显数量 futures 的 [`join3`](https://docs.rs/futures/latest/futures/future/fn.join3.html)、[`join4`](https://docs.rs/futures/latest/futures/future/fn.join4.html) 和 [`join5`](https://docs.rs/futures/latest/futures/future/fn.join5.html)，以及用于 join futures 集合的 [join_all](https://docs.rs/futures/latest/futures/future/fn.join_all.html)（以及每个的 `try_` 变体）。

[Futures-concurrency](https://docs.rs/futures-concurrency/latest) 也提供了 join（和 try_join）的功能。在 futures-concurrency 风格中，这些操作是 futures 组（如元组、`Vec` 或数组）上的 trait 方法。例如，要 join 两个 futures，你会写 `(fut1, fut2).join().await`（注意这里 `await` 是显式的）。

如果你希望 join 的 futures 集合动态变化（例如，随着输入通过网络进入而创建新的 futures），或者你希望在结果完成时而不是所有 futures 完成时获得结果，那么你需要使用 streams 和 [`FuturesUnordered`](https://docs.rs/futures/latest/futures/stream/struct.FuturesUnordered.html) 或 [`FuturesOrdered`](https://docs.rs/futures/latest/futures/stream/struct.FuturesOrdered.html) 功能。我们将在 [streams](streams.md) 章节中介绍这些内容。


[^into]: 表达式必须具有实现 `IntoFuture` 的类型。表达式由宏求值并转换为 future。即，它们实际上不必求值为 future，而是可以转换为 future 的东西，但这只是一个很小的区别。表达式本身在任何结果 futures 执行之前按顺序求值。


## Race/select

Joining futures 的对应物是 racing 它们（又名 selecting 它们）。使用 race/select，futures 并发执行，但不是等待所有 futures 完成，我们只等待第一个完成，然后取消其他的。虽然这听起来类似于 joining，但它明显更有趣（有时也更容易出错），因为现在我们必须推理取消。

这是一个使用 Tokio 的 [`select`](https://docs.rs/tokio/latest/tokio/macro.select.html) 宏的例子：

```rust,norun
async fn main() {
  select! {
    result = do_a_thing() => {
      println!("computation completed and returned {result}");
    }
    _ = timeout() => {
      println!("computation timed-out");
    }
  }
}
```

你会注意到事情已经比 `join` 宏更有趣了，因为我们在 `select` 宏中处理 futures 的结果。它看起来有点像 `match` 表达式，但是使用 `select`，所有分支都并发运行，并且首先完成的分支的主体将使用其结果执行（其他分支不执行，futures 通过 `drop` 取消）。在示例中，`do_a_thing` 和 `timeout` 并发执行，第一个完成的将执行其块（即，只有一个 `println` 会运行），另一个 future 将被取消。与 `join` 宏一样，awaiting futures 是隐式的。

Tokio 的 `select` 宏支持许多功能：

- 模式匹配：每个分支 `=` 左侧的语法可以是一个模式，只有当 future 的结果与模式匹配时才会执行该块。如果模式不匹配，则不再轮询该 future（但其他 futures 会被轮询）。这对于可选返回值的 futures 很有用，例如 `Some(x) = do_a_thing() => { ... }`。
- `if` guards：每个分支可以有一个 `if` guard。当 `select` 宏运行时，在求值每个表达式以产生 future 后，会对 `if` guard 进行求值，只有当 guard 为 true 时才会轮询 future。例如，`x = = do_a_thing() if false => { ... }` 将永远不会被轮询。请注意，`if` guard 在轮询期间不会重新求值，仅在宏初始化时求值。
- `else` 分支：`select` 可以有一个 `else` 分支 `else => { ... }`，如果所有 futures 都已停止且没有块被执行，则执行此分支。如果在没有 `else` 分支的情况下发生这种情况，那么 `select` 将恐慌。

`select!` 宏的值是执行分支的值（就像 `match` 一样），因此所有分支必须具有相同的类型。例如，如果我们想在 `select` 之外使用上面示例的结果，我们会这样写

```rust,norun
async fn main() {
  let result = select! {
    result = do_a_thing() => {
      Some(result)
    }
    _ = timeout() => {
      None
    }
  };

  // 使用 `result`
}
```

与 `join!` 一样，`select!` 不以任何特殊方式处理 `Result`（除了前面提到的模式匹配），如果一个分支以错误完成，那么所有其他分支都将被取消，并且错误将用作 select 的结果（就像分支成功完成一样）。

`select` 宏本质上使用取消，所以如果你试图在程序中避免取消，你必须避免 `select!`。事实上，`select` 通常是异步程序中取消的主要来源。正如[别处](../part-reference/cancellation.md)所讨论的，取消有许多微妙的问题，可能导致错误。特别是，请注意 `select` 通过简单地丢弃 futures 来取消它们。这不会通知被丢弃的 future 或触发任何取消令牌等。

`select!` 通常用于循环中处理 streams 或其他 futures 序列。这增加了一层额外的复杂性和出错的机会。在我们在循环的每次迭代中创建一个新的、独立的 future 的简单情况下，事情并没有复杂多少。然而，这很少是所需要的。通常我们希望在迭代之间保留一些状态。在循环中将 `select` 与 streams 一起使用是很常见的，其中循环的每次迭代处理来自 stream 的一个结果。例如：

```rust,norun
async fn main() {
  let mut stream = ...;

  loop {
    select! {
      result = stream.next() => {
        match result {
          Some(x) => println!("received: {x}"),
          None => break,
        }
      }
      _ = timeout() => {
        println!("time out!");
        break;
      }
    }
  }
}
```

在这个例子中，我们从 `stream` 读取值并打印它们，直到没有剩余值或等待结果超时。在超时情况下，stream 中剩余的数据会发生什么取决于 stream 的实现（它可能会丢失！或重复！）。这是一个例子，说明为什么面对取消时的行为可能很重要（而且棘手）。

我们可能希望在迭代之间重用 future，而不仅仅是 stream。例如，我们可能希望与超时 future 竞争，其中超时适用于所有迭代，而不是为每次迭代应用新的超时。这可以通过在循环外部创建 future 并引用它来实现：

```rust,norun
async fn main() {
  let mut stream = ...;
  let mut timeout = timeout();

  loop {
    select! {
      result = stream.next() => {
        match result {
          Some(x) => println!("received: {x}"),
          None => break,
        }
      }
      // 创建对 `timeout` 的引用而不是移动它。
      _ = &mut timeout => {
        println!("time out!");
        break;
      }
    }
  }
}
```

当在循环中使用 `select!` 以及在 `select!` 外部创建的 futures 或 streams 时，有几个重要的细节。这是 `select` 工作方式的根本后果，所以我将通过逐步介绍 `select` 的细节来介绍它们，以上一个例子中的 `timeout` 为例。

- `timeout` 在循环外部创建，并初始化为倒计时一段时间。
- 在循环的每次迭代中，`select` 创建对 `timeout` 的引用，但不改变其状态。
- 当 `select` 执行时，它轮询 `timeout`，如果有剩余时间，它将返回 `Pending`，当时间流逝时返回 `Ready`，此时执行其块。

在上面的例子中，当 `timeout` 准备好时，我们 `break` 跳出循环。但是如果我们不这样做呢？在这种情况下，`select` 只会再次轮询 `timeout`，`Future` [文档](https://doc.rust-lang.org/std/future/trait.Future.html#tymethod.poll) 说这不应该发生！`select` 对此无能为力，它没有任何状态（在迭代之间）来决定是否应该轮询 `timeout`。根据 `timeout` 的编写方式，这可能会导致恐慌、逻辑错误或某种崩溃。

你可以通过几种方式防止此类错误：

- 使用 [fused](futures.md#fusing) [future](https://docs.rs/futures/latest/futures/future/trait.FutureExt.html#method.fuse) 或 [stream](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html#method.fuse)，以便重新轮询是安全的。
- 确保你的代码结构使得 futures 永远不会被重新轮询，例如，通过跳出循环（如前面的例子），或使用 `if` guard。

现在，让我们考虑 `&mut timeout` 的类型。让我们假设 `timeout()` 返回一个实现 `Future` 的类型，它可能是来自异步函数的匿名类型，也可能是像 `Timeout` 这样的命名类型。让我们假设后者，因为它使例子更容易（但逻辑适用于任何一种情况）。鉴于 `Timeout` 实现了 `Future`，`&mut Timeout` 会实现 `Future` 吗？不一定！有一个 [blanket `impl`](https://doc.rust-lang.org/std/future/trait.Future.html#impl-Future-for-%26mut+F) 使这成为现实，但前提是 `Timeout` 实现 `Unpin`。并非所有 futures 都是这种情况，所以通常你会像上一个例子那样编写代码时遇到类型错误。不过，通过使用 `pin` 宏很容易修复这种错误，例如 `let mut timeout = pin!(timeout());`

在循环中使用 `select` 进行取消是微妙错误的丰富来源。这些通常发生在 future 包含一些涉及某些数据但不包含数据本身的状态的地方。当 future 因取消而被丢弃时，该状态丢失，但底层数据未更新。这可能导致数据丢失或被多次处理。


### 替代方案

Futures.rs 有自己的 [`select` 宏](https://docs.rs/futures/latest/futures/macro.select.html)，futures-concurrency 有一个 [Race trait](https://docs.rs/futures-concurrency/latest/futures_concurrency/future/trait.Race.html)，它们是 Tokio `select` 宏的替代方案。它们都具有并发竞争多个 futures 的相同核心语义，处理第一个的结果并取消其他的，但它们具有不同的语法并且在细节上有所不同。

Futures.rs 的 `select` 表面上类似于 Tokio 的；总结一下区别，在 futures.rs 版本中：

- Futures 必须始终是 fused（通过类型检查强制执行）。
- `select` 有 `default` 和 `complete` 分支，而不是 `else` 分支。
- `select` 不支持 `if` guards。

Futures-concurrency 的 `Race` 具有非常不同的语法，类似于它的 `join` 版本，例如 `(future_a, future_b).race().await`（它适用于 `Vec` 和数组以及元组）。语法不如宏灵活，但非常适合大多数异步代码。请注意，如果你在循环中使用 `race`，你仍然可能会遇到与 `select` 相同的问题。

与 `join` 一样，派生任务并让它们并行执行通常是使用 `select` 的一个很好的替代方案。但是，在第一个任务完成后取消剩余任务需要一些额外的工作。这可以使用通道或取消令牌来完成。在任何一种情况下，取消都需要被取消的任务采取一些行动，这意味着任务可以进行一些整理或其他优雅的关闭。

`select` 的一个常见用途（尤其是在循环内部）是处理 streams。有一些 stream 组合器方法可以替代 `select` 的某些用途。例如，futures-concurrency 中的 [`merge`](https://docs.rs/futures-concurrency/latest/futures_concurrency/stream/trait.Merge.html) 是将多个 streams 合并在一起的一个很好的替代方案。


## 结语

在本节中，我们讨论了两种并发运行 futures 组的方法。Joining futures 意味着等待它们全部完成；selecting (aka racing) futures 意味着等待第一个完成。与派生任务相比，这些组合不使用并行。

`join` 和 `select` 都对预先知道的 futures 集合进行操作（通常在编写程序时，而不是在运行时）。有时，要组合的 futures 并不是预先知道的——futures 必须在执行时添加到组合 futures 集合中。为此，我们需要 [streams](streams.md)，它们有自己的组合操作。

值得重申的是，虽然这些组合运算符功能强大且富有表现力，但使用任务和派生通常更容易且更合适：并行通常是可取的，你不太可能遇到关于取消或阻塞的错误，并且资源分配通常更公平（或至少更简单）且更可预测。
