# 最终项目：使用异步 Rust 构建并发 Web 服务器
在本章中，我们将使用异步 Rust 修改 Rust 书籍中的 [单线程 Web 服务器](https://doc.rust-lang.org/book/ch20-01-single-threaded.html)，以并发地处理请求。
## 回顾
这是课程结束时的代码。

`src/main.rs`:
```rust
{{#include ../../examples/09_01_sync_tcp_server/src/main.rs}}
```

`hello.html`:
```html
{{#include ../../examples/09_01_sync_tcp_server/hello.html}}
```

`404.html`:
```html
{{#include ../../examples/09_01_sync_tcp_server/404.html}}
```

如果你使用 `cargo run` 运行服务器并在浏览器中访问 `127.0.0.1:7878`，你将收到来自 Ferris 的友好问候！
