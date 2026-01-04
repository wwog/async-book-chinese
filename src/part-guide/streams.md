# 异步迭代器 (前称 Streams)

- Stream 作为异步迭代器或多个 futures
- 进行中 (WIP)
  - 当前状态
  - futures 和 Tokio Stream traits
  - nightly trait
- 像同步迭代器一样是惰性的
- pinning 和 streams (前向引用到 pinning 章节)
- fused streams (融合流)

## 消费异步迭代器

- while let 配合 async next
- for_each, for_each_concurrent
- collect
- into_future, buffered

## Stream 组合器

- 接受 future 而不是闭包
- 一些组合器示例
- 无序变体
- StreamGroup

### join/select/race 与 streams

- 循环中使用 select 的风险
- fusing (融合)
- 与仅使用 futures 的区别
- 这些的替代方案
  - Stream::merge 等

## 实现异步迭代器

- 实现 trait
- 实用性和工具函数
- async_iter stream 宏

## Sinks

- https://docs.rs/futures/latest/futures/sink/index.html

## 未来工作

- 当前状态
  - https://rust-lang.github.io/rfcs/2996-async-iterator.html
- async next vs poll
- 异步迭代语法
- (异步) 生成器
- lending iterators (借用迭代器)

