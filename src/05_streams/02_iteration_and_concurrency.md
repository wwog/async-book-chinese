# 迭代和并发

与同步 `Iterator` 类似，有许多不同的方法可以迭代和处理 `Stream` 中的值。有组合器风格的方法，如 `map`、`filter` 和 `fold`，以及它们在出错时提前退出的表亲 `try_map`、`try_filter` 和 `try_fold`。

不幸的是，`for` 循环不能用于 `Stream`，但对于命令式风格的代码，可以使用 `while let` 和 `next`/`try_next` 函数：

```rust,edition2018,ignore
{{#include ../../examples/05_02_iteration_and_concurrency/src/lib.rs:nexts}}
```

然而，如果我们一次只处理一个元素，我们可能会放弃并发的机会，这毕竟是我们首先编写异步代码的原因。要并发处理流中的多个项目，请使用 `for_each_concurrent` 和 `try_for_each_concurrent` 方法：

```rust,edition2018,ignore
{{#include ../../examples/05_02_iteration_and_concurrency/src/lib.rs:try_for_each_concurrent}}
```
