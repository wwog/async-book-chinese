# 递归

在内部，`async fn` 创建一个包含每个被 `.await` 的子 `Future` 的状态机类型。这使得递归 `async fn` 有点棘手，因为生成的状态机类型必须包含其自身：

```rust,edition2018
# async fn step_one() { /* ... */ }
# async fn step_two() { /* ... */ }
# struct StepOne;
# struct StepTwo;
// 这个函数：
async fn foo() {
    step_one().await;
    step_two().await;
}
// 生成像这样的类型：
enum Foo {
    First(StepOne),
    Second(StepTwo),
}

// 所以这个函数：
async fn recursive() {
    recursive().await;
    recursive().await;
}

// 生成像这样的类型：
enum Recursive {
    First(Recursive),
    Second(Recursive),
}
```

这是行不通的——我们创建了一个无限大小的类型！
编译器会抱怨：

```
error[E0733]: recursion in an async fn requires boxing
 --> src/lib.rs:1:1
  |
1 | async fn recursive() {
  | ^^^^^^^^^^^^^^^^^^^^
  |
  = note: a recursive `async fn` call must introduce indirection such as `Box::pin` to avoid an infinitely sized future
```

为了允许这样做，我们必须使用 `Box` 引入间接层。

在 Rust 1.77 之前，由于编译器限制，仅仅将 `recursive()` 的调用包装在 `Box::pin` 中是不够的。为了使其工作，我们必须将 `recursive` 变成一个非 `async` 函数，该函数返回一个 `.boxed()` `async` 块：

```rust,edition2018
{{#include ../../examples/07_05_recursion/src/lib.rs:example}}
```

在较新版本的 Rust 中，[该编译器限制已被解除]。

自 Rust 1.77 以来，对带有分配间接层的 `async fn` 中的递归支持 [变得稳定]，因此只要递归调用使用某种形式的间接层来避免函数状态的无限大小，就是允许的。

这意味着像这样的代码现在可以工作：

```rust,edition2021
{{#include ../../examples/07_05_recursion/src/lib.rs:example_pinned}}
```

[变得稳定]: https://blog.rust-lang.org/2024/03/21/Rust-1.77.0.html#support-for-recursion-in-async-fn
[该编译器限制已被解除]: https://github.com/rust-lang/rust/pull/117703/
