# 定时器和信号处理

## 时间和定时器

- 运行时集成，不要使用 thread::sleep 等
- std Instant 和 Duration
- sleep (睡眠)
- interval (间隔)
- timeout (超时)
  - 特殊 future vs select/race

## 信号处理

- 什么是信号处理，为什么它是一个异步问题？
- 非常依赖于操作系统
- 参阅 Tokio 文档
