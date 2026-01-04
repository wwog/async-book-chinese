# 更多 async/await 主题

本章涵盖了关于 `async` 和 `await` 的一些更高级的主题。

## 单元测试

单元测试是 Rust 开发的重要组成部分。然而，由于 `async` 函数返回的是一个 Future 而不是直接执行，因此你不能像普通函数那样直接测试它们。

```rust,norun
#[test]
async fn test_foo() {
    // ...
}
```

如果你尝试这样做，编译器会报错，因为它不知道如何执行这个 Future。为了测试 `async` 函数，你需要一个运行时来执行 Future。大多数异步运行时都提供了用于测试的宏，例如 `tokio::test` 或 `async_std::test`。

```rust,norun
#[tokio::test]
async fn test_foo() {
    // ...
}
```

这些宏会将你的 `async` 测试函数包装在一个同步测试函数中，该同步函数会初始化运行时并阻塞直到 Future 完成。

如果你不想依赖特定的运行时，你可以手动创建一个运行时并在其中运行你的测试，或者使用像 `futures::executor::block_on` 这样的简单执行器。

```rust,norun
#[test]
fn test_foo() {
    futures::executor::block_on(async {
        // ...
    });
}
```

## 阻塞与取消

在异步代码中，长时间运行的同步操作（阻塞操作）是一个大问题。因为异步运行时通常使用少量的线程来处理大量的任务，如果其中一个任务阻塞了线程，那么该线程上的其他任务就无法执行。这会导致整个应用程序的性能下降，甚至死锁。

常见的阻塞操作包括：
- 标准库的同步 I/O（如 `std::fs`，`std::net`）
- 繁重的 CPU 计算
- 同步互斥锁（如 `std::sync::Mutex`）的长时间持有

你应该尽量避免在异步代码中使用这些操作。对于 I/O，应使用异步版本（如 `tokio::fs`，`tokio::net`）。对于 CPU 密集型任务，可以使用 `spawn_blocking` 将其卸载到专用线程池中。

取消是异步编程中的另一个重要概念。Future 是惰性的，只有在被轮询（poll）时才会执行。这意味着如果你停止轮询一个 Future（例如，通过丢弃它），它的执行就会停止。这被称为“取消”。

当你使用 `select!` 宏或超时机制时，未完成的分支会被取消（丢弃）。因此，编写能够安全取消的异步代码非常重要。这意味着你的 Future 应该在被丢弃时能够正确地清理资源，并且不应该在取消时留下不一致的状态。

## Async 块

`async` 块允许你在普通函数中创建一个 Future。这在你想在同步上下文中启动异步操作，或者想更精细地控制 Future 的创建时非常有用。

```rust,norun
fn foo() {
    let future = async {
        println!("Hello from async block");
    };
    // ...
}
```

`async` 块可以捕获环境中的变量。与闭包类似，你可以使用 `move` 关键字将所有权转移到块中。

```rust,norun
fn foo() {
    let x = 5;
    let future = async move {
        println!("x is {}", x);
    };
}
```

在 `async` 块中使用控制流（如 `return`，`break`，`continue`）时要小心。`return` 语句会从 `async` 块返回，而不是从包含它的函数返回。

```rust,norun
async fn foo() {
    // ...
    let x = async {
        if ... {
            return 1; // 返回 1 作为 async 块的结果
        }
        2
    }.await;
    // ...
}
```

如果你想从 `async` 块中跳出循环，你需要注意 `break` 和 `continue` 只能作用于 `async` 块内部的循环。

```rust,norun
loop {
    if ... {
        // ok
        continue;
    }

    async {
        if ... {
            // not ok
            // continue;

            // ok - 继续执行 `loop` 的下一次迭代，但请注意，如果 async 块之后还有代码，
            // 那些代码将会被执行。
            return;
        }
    }.await
}
```

要实现 `break`，你需要测试块的返回值（一种常见的习惯用法是使用 [`ControlFlow`](https://doc.rust-lang.org/std/ops/enum.ControlFlow.html) 作为块的值，这也允许使用 `?`）。

同样，`async` 块内的 `?` 会在出现错误时终止 Future 的执行，导致 `await` 的块取值为该错误，但不会退出包含它的函数（就像普通块中的 `?` 那样）。你需要 `await` 之后再加一个 `?` 来实现这一点：

```rust,norun
async {
    let x = foo()?;   // 这个 `?` 只退出 async 块，不退出外层函数。
    consume(x);
    Ok(())
}.await?
```

令人烦恼的是，这通常会使编译器感到困惑，因为（与函数不同）`async` 块的“返回”类型没有显式声明。你可能需要在变量上添加类型注解，或者使用 turbofish 语法来指定类型，例如在上面的例子中使用 `Ok::<_, MyError>(())` 而不是 `Ok(())`。

返回 `async` 块的函数与 `async` 函数非常相似。写 `async fn foo() -> ... { ... }` 大致等同于 `fn foo() -> ... { async { ... } }`。事实上，从调用者的角度来看，它们是等价的，从一种形式改为另一种形式不是破坏性变更。此外，在实现异步 trait 时，你可以用一种形式覆盖另一种（见下文）。但是，你必须调整类型，在 `async` 块版本中显式写出 `Future`：`async fn foo() -> Foo` 变为 `fn foo() -> impl Future<Output = Foo>`（你可能还需要显式指定其他边界，例如 `Send` 和 `'static`）。

通常你会更喜欢 `async` 函数版本，因为它更简单、更清晰。然而，`async` 块版本更灵活，因为你可以在调用函数时执行一些代码（通过写在 `async` 块之外），并在结果被 await 时执行另一些代码（写在 `async` 块内部）。

## Async 闭包

- 闭包
  - 即将推出 (https://github.com/rust-lang/rust/pull/132706, https://blog.rust-lang.org/inside-rust/2024/08/09/async-closures-call-for-testing.html)
  - 闭包中的 async 块 vs async 闭包

## 生命周期和借用

- 上面提到了静态生命周期
- Future 上的生命周期边界 (`Future + '_` 等)
- 跨 await 点的借用
- 我不知道，我确信 async 函数还有更多生命周期问题 ...

## Future 上的 `Send + 'static` 边界

- 为什么需要它们，多线程运行时
- spawn local 以避免它们
- 是什么让 async fn 变为 `Send + 'static` 以及如何修复相关 bug

## Async traits

- 语法
  - `Send + 'static` 问题及解决方法
    - trait_variant
    - 显式 future
    - 返回类型标记 (https://blog.rust-lang.org/inside-rust/2024/09/26/rtn-call-for-testing.html)
- 覆盖
  - 方法的 future vs async 标记
- 对象安全性
- 捕获规则 (https://blog.rust-lang.org/2024/09/05/impl-trait-capture-rules.html)
- 历史和 async-trait crate

## 递归

- 允许（相对较新），但需要一些显式的装箱（boxing）
  - 前向引用 futures，pinning
  - https://rust-lang.github.io/async-book/07_workarounds/04_recursion.html
  - https://blog.rust-lang.org/2024/03/21/Rust-1.77.0.html#support-for-recursion-in-async-fn
  - async-recursion 宏 (https://docs.rs/async-recursion/latest/async_recursion/)

