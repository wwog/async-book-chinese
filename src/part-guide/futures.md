# Futures

我们在前面的章节中已经讨论了很多关于 futures 的内容；它们是 Rust 异步编程故事的关键部分！在本章中，我们将深入探讨 futures 是什么以及它们如何工作的细节，以及一些直接处理 futures 的库。

## `Future` 和 `IntoFuture` traits

- Future
  - Output 关联类型
  - 这里没有真正的细节，轮询在下一节，参考关于 Pin、执行器/唤醒器的高级部分
- IntoFuture
  - 用法 - 一般用法，在 await 中，异步构建器模式（使用的优缺点）
- 装箱 futures，`Box<dyn Future>` 以及它过去如何常见和必要，但现在大多不是了，除了递归等。

## 轮询 (Polling)

- 它是什么以及谁在做，Poll 类型
  - ready 是最终状态
- 它如何与 await 连接
- drop = 取消
  - 对于 futures 以及因此对于任务
  - 对一般异步编程的影响
  - 参考关于取消安全的章节

### Fusing

## futures-rs crate

- 历史和目的
  - 见 streams 章节
  - 编写执行器或其他低级 futures 内容的助手
    - pinning 和 boxing
  - 作为部分运行时的执行器（见参考中的替代运行时）
- TryFuture
- 便利 futures：pending, ready, ok/err 等
- FutureExt 上的组合器函数
- Tokio 内容的替代品
  - 函数
  - IO traits

## futures-concurrency crate

https://docs.rs/futures-concurrency/latest/futures_concurrency/


