# 运行时和运行时问题

## 运行异步代码

- 显式启动 vs async main
- tokio 上下文概念
- block_on
- 代码中反映的运行时 (Runtime, Handle)
- 运行时关闭

## 线程和任务

- 默认工作窃取，多线程
  - 重温 Send + 'static 边界
- yield (让出执行权)
- spawn-local (本地生成)
- spawn-blocking (阻塞生成) (回顾), block-in-place (原地阻塞)
- tokio 特有的关于让出给其他线程、本地 vs 全局队列等内容

## 配置选项

- 线程池大小
- 单线程，每核一线程等

## 替代运行时

- 为什么你会想要使用不同的运行时或实现自己的运行时
- 高层设计中存在什么样的变体
- 前向引用到高级章节
