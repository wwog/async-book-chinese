# `join!`

`futures::join` 宏使得等待多个不同的 future 完成成为可能，同时并发执行它们。

## `join!`

当执行多个异步操作时，很容易简单地按顺序 `.await` 它们：

```rust,edition2018,ignore
{{#include ../../examples/06_02_join/src/lib.rs:naiive}}
```

然而，这将比必要的慢，因为它直到 `get_book` 完成后才会开始尝试 `get_music`。在其他一些语言中，future 是在后台运行至完成的，因此可以通过首先调用每个 `async fn` 来启动 future，然后等待它们两个来并发运行两个操作：

```rust,edition2018,ignore
{{#include ../../examples/06_02_join/src/lib.rs:other_langs}}
```

然而，Rust future 在被主动 `.await` 之前不会做任何工作。这意味着上面的两个代码片段都将按顺序运行 `book_future` 和 `music_future`，而不是并发运行它们。要正确地并发运行这两个 future，请使用 `futures::join!`：

```rust,edition2018,ignore
{{#include ../../examples/06_02_join/src/lib.rs:join}}
```

`join!` 返回的值是一个包含传入的每个 `Future` 输出的元组。

## `try_join!`

对于返回 `Result` 的 future，考虑使用 `try_join!` 而不是 `join!`。由于 `join!` 只有在所有子 future 都完成后才完成，即使其中一个子 future 返回了 `Err`，它也会继续处理其他 future。

与 `join!` 不同，如果其中一个子 future 返回错误，`try_join!` 将立即完成。

```rust,edition2018,ignore
{{#include ../../examples/06_02_join/src/lib.rs:try_join}}
```

请注意，传递给 `try_join!` 的 future 必须都具有相同的错误类型。考虑使用 `futures::future::TryFutureExt` 中的 `.map_err(|e| ...)` 和 `.err_into()` 函数来统一错误类型：

```rust,edition2018,ignore
{{#include ../../examples/06_02_join/src/lib.rs:try_join_map_err}}
```
