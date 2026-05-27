---
type: wiki
updated: 2026-04-24
tags:
  - Java
  - 并发
---

# Java并发

当前并发资料覆盖了从线程基础到并发工具的主干内容，已经可以支撑一轮比较系统的面试复习和知识梳理。

## 正文

- 第一层是线程/进程、多线程、上下文切换、死锁等基础概念。
- 第二层是 JMM（Java 内存模型，强调多线程的**可见性、原子性、有序性**）、`volatile`（禁止指令重排与保证可见性）、`synchronized` 锁升级过程（偏向锁 -> 轻量级锁 -> 重量级锁）、`ReentrantLock`（显式锁，支持响应中断、公平锁、多条件变量）等并发语义和同步机制。
- 第三层是并发常用工具的深度机制：
  - **ThreadLocal 内存泄漏**：ThreadLocalMap 中的 Key 为 ThreadLocal 的弱引用，而 Value 是强引用。如果 ThreadLocal 被回收但线程依然存活（如线程池中复用线程），Value 就无法被回收造成泄漏。解决方案：每次使用完务必手动调用 `remove()` 方法。
  - **线程池（ThreadPoolExecutor）**：
    - **7 个核心参数**：`corePoolSize`（核心线程数）、`maximumPoolSize`（最大线程数）、`keepAliveTime`（存活时间）、`unit`（时间单位）、`workQueue`（工作队列）、`threadFactory`（线程工厂）、`handler`（拒绝策略）。
    - **4 种拒绝策略**：`AbortPolicy`（丢弃并抛异常，默认）、`CallerRunsPolicy`（调用者运行）、`DiscardPolicy`（直接丢弃）、`DiscardOldestPolicy`（丢弃最老任务）。
    - **工作流程**：提交任务 -> 核心线程未满则创建线程 -> 满了则放入工作队列 -> 队列满了且未达最大线程数则创建非核心线程 -> 达到最大线程数且队列满则执行拒绝策略。
  - **AQS 同步器**：AbstractQueuedSynchronizer 是并发锁的基石。内部采用一个 **state 状态变量**（volatile修饰，代表锁资源）以及一个双向链表组成的 **FIFO 队列**。多线程通过 CAS 抢占 state 状态，抢占失败则被封装为 Node 节点加入双向队列中阻塞等待唤醒。
- 这部分知识和操作系统、JVM、Spring 异步机制有明显交叉，应当交叉阅读。

## 相关链接

- [Java基础](wiki/topics/Java基础.md)
- [JVM](wiki/topics/JVM.md)
- [操作系统](wiki/topics/操作系统.md)
- [Spring生态](wiki/topics/Spring生态.md)

## 来源

- [Java并发来源整理](raw/sources/Java并发来源整理.md)
