# Trait 中的 `async`

目前，在 Rust 的稳定版本中，`async fn` 不能在 trait 中使用。
自 2022 年 11 月 17 日起，async-fn-in-trait 的 MVP 版本已在编译器工具链的 nightly 版本中可用，[详情请见此处](https://blog.rust-lang.org/inside-rust/2022/11/17/async-fn-in-trait-nightly.html)。

在此期间，对于稳定工具链，可以使用 [crates.io 上的 async-trait crate](https://github.com/dtolnay/async-trait) 作为变通方法。

请注意，使用这些 trait 方法将导致每次函数调用进行一次堆分配。对于绝大多数应用程序来说，这不是一个显著的成本，但在决定是否在预计每秒调用数百万次的低级函数的公共 API 中使用此功能时，应予以考虑。

最新更新：https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits.html
