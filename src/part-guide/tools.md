# 异步编程工具

- 为什么我们需要专门的异步工具
- 是否有其他工具需要涵盖
  - loom

## 监控 (Monitoring)

- [Tokio console](https://github.com/tokio-rs/console)

## 追踪和日志 (Tracing and logging)

- 异步追踪的问题
- tracing crate (https://github.com/tokio-rs/tracing)

## 调试 (Debugging)

- 理解异步回溯 (RUST_BACKTRACE 和调试器中)
- 调试异步代码的技术
- 使用 Tokio console 进行调试
- 调试器支持 (WinDbg?)

## 性能分析 (Profiling)

- 异步如何搞乱火焰图
- 如何分析异步 IO
- 深入了解运行时
  - Tokio metrics
