# 取消和取消安全性

内部 vs 外部取消
线程 vs futures
  drop = cancel (丢弃即取消)
  仅在 await 点
  有用的特性
  仍然有些突然和令人惊讶
其他取消机制
  abort (中止)
  cancellation tokens (取消令牌)

## 取消安全性 (Cancellation safety)

不是内存安全问题或竞争条件
  数据丢失或其他逻辑错误
不同的定义/名称
  tokio 的定义
  通用定义/halt safety (停机安全性)
  应用复制 future 的想法
简单的数据丢失
恢复 (Resumption)
循环中使用 select 或类似机制的问题
作为根本原因的状态在 future 和上下文之间的分割


