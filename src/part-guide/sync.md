# 通道、锁和同步

关于同步原语的运行时特定性的说明

为什么我们需要异步原语而不是使用同步原语

## 通道 (Channels)

- 基本上与标准库的相同，但是需要 await
  - 在任务之间通信（同一线程或不同线程）
- one shot (单次发送)
- mpsc (多生产者单消费者)
- 其他通道
- 有界和无界通道

## 锁 (Locks)

- 异步 Mutex
  - 对比 std::Mutex - 可以跨 await 点持有（在 guard 中借用 mutex，guard 是 Send 的，调度器感知的？还是仅仅因为 lock 是异步的？），lock 是异步的（等待锁可用时不会阻塞线程）
    - 甚至有一个 clippy lint 用于检查跨 await 持有 guard (https://rust-lang.github.io/rust-clippy/master/index.html#await_holding_lock)
  - 因为可以跨 await 持有，所以开销更大
    - 如果可以，请使用 std::Mutex
      - 可以使用 try_lock 或者预期 mutex 不会有争用
  - yield 时锁不会神奇地被丢弃（这正是锁的意义所在！）
  - 跨 await 持有 mutex 导致的死锁
    - 任务死锁，但其他任务可以继续进行，所以在进程统计/工具/操作系统中可能看起来不像死锁
    - 通常的建议 - 限制范围，最小化锁，锁排序，首选替代方案
  - 没有 mutex poisoning (互斥锁中毒)
  - lock_owned
  - blocking_lock
    - 不能在异步中使用
  - 适用于其他锁（上面的内容应该移到具体讨论 mutex 之前吗？可能应该）
- RWLock (读写锁)
- Semaphore (信号量)
- yielding (让出)

## 其他同步原语

- notify, barrier (屏障)
- OnceCell
- atomics (原子操作)
