# `Send` 近似

一些 `async fn` 状态机可以安全地跨线程发送，而另一些则不能。`async fn` `Future` 是否为 `Send` 取决于是否在 `.await` 点持有非 `Send` 类型。编译器会尽力近似判断何时值可能在 `.await` 点被持有，但这种分析在今天的一些地方过于保守。

例如，考虑一个简单的非 `Send` 类型，也许是一个包含 `Rc` 的类型：

```rust
use std::rc::Rc;

#[derive(Default)]
struct NotSend(Rc<()>);
```

`NotSend` 类型的变量可以作为临时变量短暂出现在 `async fn` 中，即使 `async fn` 返回的结果 `Future` 类型必须是 `Send`：

```rust,edition2018
# use std::rc::Rc;
# #[derive(Default)]
# struct NotSend(Rc<()>);
async fn bar() {}
async fn foo() {
    NotSend::default();
    bar().await;
}

fn require_send(_: impl Send) {}

fn main() {
    require_send(foo());
}
```

然而，如果我们更改 `foo` 以将 `NotSend` 存储在变量中，此示例将不再编译：

```rust,edition2018
# use std::rc::Rc;
# #[derive(Default)]
# struct NotSend(Rc<()>);
# async fn bar() {}
async fn foo() {
    let x = NotSend::default();
    bar().await;
}
# fn require_send(_: impl Send) {}
# fn main() {
#    require_send(foo());
# }
```

```
error[E0277]: `std::rc::Rc<()>` cannot be sent between threads safely
  --> src/main.rs:15:5
   |
15 |     require_send(foo());
   |     ^^^^^^^^^^^^ `std::rc::Rc<()>` cannot be sent between threads safely
   |
   = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::rc::Rc<()>`
   = note: required because it appears within the type `NotSend`
   = note: required because it appears within the type `{NotSend, impl std::future::Future, ()}`
   = note: required because it appears within the type `[static generator@src/main.rs:7:16: 10:2 {NotSend, impl std::future::Future, ()}]`
   = note: required because it appears within the type `std::future::GenFuture<[static generator@src/main.rs:7:16: 10:2 {NotSend, impl std::future::Future, ()}]>`
   = note: required because it appears within the type `impl std::future::Future`
   = note: required because it appears within the type `impl std::future::Future`
note: required by `require_send`
  --> src/main.rs:12:1
   |
12 | fn require_send(_: impl Send) {}
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
```

这个错误是正确的。如果我们将 `x` 存储在变量中，它直到 `.await` 之后才会被丢弃，此时 `async fn` 可能正在不同的线程上运行。由于 `Rc` 不是 `Send`，允许它跨线程传输将是不安全的。对此的一个简单解决方案是在 `.await` 之前 `drop` `Rc`，但不幸的是，这在今天不起作用。

为了成功解决此问题，你可能必须引入一个封装任何非 `Send` 变量的块作用域。这使得编译器更容易判断这些变量不会跨越 `.await` 点存活。

```rust,edition2018
# use std::rc::Rc;
# #[derive(Default)]
# struct NotSend(Rc<()>);
# async fn bar() {}
async fn foo() {
    {
        let x = NotSend::default();
    }
    bar().await;
}
# fn require_send(_: impl Send) {}
# fn main() {
#    require_send(foo());
# }
```
