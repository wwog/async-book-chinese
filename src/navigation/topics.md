# 主题索引

## 并发与并行

- [简介](../part-guide/concurrency.md#concurrency-and-parallelism)
- [使用 `spawn` 并行运行异步任务](../part-guide/async-await.md#spawning-tasks)
- [使用 `join` 和 `select` 并发运行 futures](../part-guide/concurrency-primitives.md)
- [混合同步和异步并发](../part-guide/io.md#other-blocking-operations)


## 正确性与安全性

- 取消
  - [简介](../part-guide/more-async-await.md#cancellation)
  - [在 `select` 和 `try_join` 中](../part-guide/concurrency-primitives.md)


## 性能

- 阻塞
  - [简介](../part-guide/more-async-await.md#blocking-and-cancellation)
  - [阻塞与非阻塞 IO](../part-guide/io.md)
  - [CPU 密集型代码](../part-guide/io.md#other-blocking-operations)


## 测试

- [单元测试语法](../part-guide/more-async-await.md#unit-tests)
