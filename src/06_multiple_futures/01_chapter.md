# 同时执行多个 Future

到目前为止，我们主要通过使用 `.await` 来执行 future，这会阻塞当前任务，直到特定的 `Future` 完成。然而，真正的异步应用程序通常需要并发执行多个不同的操作。

在本章中，我们将介绍一些同时执行多个异步操作的方法：

- `join!`: 等待所有 future 完成
- `select!`: 等待多个 future 中的一个完成
- Spawning: 创建一个顶级任务，该任务在后台运行 future 直至完成
- `FuturesUnordered`: 一组 future，产生每个子 future 的结果
