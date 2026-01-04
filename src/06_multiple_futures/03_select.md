# `select!`

`futures::select` 宏同时运行多个 future，允许用户在任何 future 完成时立即响应。

```rust,edition2018
{{#include ../../examples/06_03_select/src/lib.rs:example}}
```

上面的函数将并发运行 `t1` 和 `t2`。当 `t1` 或 `t2` 完成时，相应的处理程序将调用 `println!`，并且函数将结束而不完成剩余的任务。

`select` 的基本语法是 `<pattern> = <expression> => <code>,`，对你想 `select` 的任意数量的 future 重复此模式。

## `default => ...` 和 `complete => ...`

`select` 还支持 `default` 和 `complete` 分支。

如果被 `select` 的 future 都没有完成，`default` 分支将运行。因此，带有 `default` 分支的 `select` 将始终立即返回，因为如果其他 future 都没有准备好，`default` 将运行。

`complete` 分支可用于处理所有被 `select` 的 future 都已完成且不再取得进展的情况。这在循环 `select!` 时通常很有用。

```rust,edition2018
{{#include ../../examples/06_03_select/src/lib.rs:default_and_complete}}
```

## 与 `Unpin` 和 `FusedFuture` 的交互

在上面的第一个例子中，你可能已经注意到我们必须对两个 `async fn` 返回的 future 调用 `.fuse()`，并使用 `pin_mut` 固定它们。这两个调用都是必要的，因为 `select` 中使用的 future 必须同时实现 `Unpin` trait 和 `FusedFuture` trait。

`Unpin` 是必要的，因为 `select` 使用的 future 不是按值获取的，而是按可变引用获取的。通过不获取 future 的所有权，未完成的 future 可以在调用 `select` 后再次使用。

同样，`FusedFuture` trait 是必需的，因为 `select` 必须在 future 完成后不再轮询它。`FusedFuture` 由跟踪其是否已完成的 future 实现。这使得在循环中使用 `select` 成为可能，只轮询尚未完成的 future。这可以在上面的例子中看到，其中 `a_fut` 或 `b_fut` 将在第二次循环时完成。因为 `future::ready` 返回的 future 实现了 `FusedFuture`，它能够告诉 `select` 不要再次轮询它。

请注意，流具有相应的 `FusedStream` trait。实现此 trait 或已使用 `.fuse()` 包装的流将从其 `.next()` / `.try_next()` 组合器产生 `FusedFuture` future。

```rust,edition2018
{{#include ../../examples/06_03_select/src/lib.rs:fused_stream}}
```

## 在 `select` 循环中使用 `Fuse` 和 `FuturesUnordered` 的并发任务

一个有点难以发现但很有用的函数是 `Fuse::terminated()`，它允许构造一个已经终止的空 future，稍后可以用需要运行的 future 填充它。

当有一个任务需要在 `select` 循环期间运行，但该任务是在 `select` 循环本身内部创建时，这很有用。

请注意 `.select_next_some()` 函数的使用。这可以与 `select` 一起使用，仅运行从流返回的 `Some(_)` 值的分支，忽略 `None`。

```rust,edition2018
{{#include ../../examples/06_03_select/src/lib.rs:fuse_terminated}}
```

当需要同时运行同一 future 的许多副本时，请使用 `FuturesUnordered` 类型。下面的例子与上面的类似，但会将 `run_on_new_num_fut` 的每个副本运行至完成，而不是在创建新副本时中止它们。它还将打印出 `run_on_new_num_fut` 返回的值。

```rust,edition2018
{{#include ../../examples/06_03_select/src/lib.rs:futures_unordered}}
```
