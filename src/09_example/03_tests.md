# 测试 TCP 服务器
让我们继续测试我们的 `handle_connection` 函数。

首先，我们需要一个 `TcpStream` 来工作。
在端到端或集成测试中，我们可能希望建立一个真正的 TCP 连接来测试我们的代码。
这样做的一种策略是在 `localhost` 端口 0 上启动一个侦听器。
端口 0 不是有效的 UNIX 端口，但它适用于测试。
操作系统将为我们选择一个开放的 TCP 端口。

相反，在这个例子中，我们将为连接处理程序编写一个单元测试，以检查是否为相应的输入返回了正确的响应。
为了保持我们的单元测试隔离和确定性，我们将用模拟替换 `TcpStream`。

首先，我们将更改 `handle_connection` 的签名以使其更易于测试。
`handle_connection` 实际上不需要 `async_std::net::TcpStream`；
它需要任何实现 `async_std::io::Read`、`async_std::io::Write` 和 `marker::Unpin` 的结构体。
更改类型签名以反映这一点允许我们传递模拟进行测试。
```rust,ignore
use async_std::io::{Read, Write};

async fn handle_connection(mut stream: impl Read + Write + Unpin) {
```

接下来，让我们构建一个实现这些 trait 的模拟 `TcpStream`。
首先，让我们实现 `Read` trait，其中包含一个方法 `poll_read`。
我们的模拟 `TcpStream` 将包含一些复制到读取缓冲区的数据，我们将返回 `Poll::Ready` 以表示读取完成。
```rust,ignore
{{#include ../../examples/09_05_final_tcp_server/src/main.rs:mock_read}}
```

我们的 `Write` 实现非常相似，尽管我们需要编写三个方法：`poll_write`、`poll_flush` 和 `poll_close`。
`poll_write` 将任何输入数据复制到模拟 `TcpStream` 中，并在完成时返回 `Poll::Ready`。
不需要做任何工作来刷新或关闭模拟 `TcpStream`，所以 `poll_flush` 和 `poll_close` 可以直接返回 `Poll::Ready`。
```rust,ignore
{{#include ../../examples/09_05_final_tcp_server/src/main.rs:mock_write}}
```

最后，我们的模拟需要实现 `Unpin`，表示其在内存中的位置可以安全地移动。
有关 pinning 和 `Unpin` trait 的更多信息，请参阅关于 pinning 的部分。
```rust,ignore
{{#include ../../examples/09_05_final_tcp_server/src/main.rs:unpin}}
```

现在我们准备测试 `handle_connection` 函数了。
在设置包含一些初始数据的 `MockTcpStream` 后，我们可以使用属性 `#[async_std::test]` 运行 `handle_connection`，类似于我们使用 `#[async_std::main]` 的方式。
为了确保 `handle_connection` 按预期工作，我们将检查是否根据其初始内容将正确的数据写入 `MockTcpStream`。
```rust,ignore
{{#include ../../examples/09_05_final_tcp_server/src/main.rs:test}}
```
