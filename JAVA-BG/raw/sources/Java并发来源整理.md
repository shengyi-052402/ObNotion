---
type: source
status: processed
source_type: clippings
created: 2026-04-24
tags:
  - Java
  - 并发
---

# Java并发来源整理

## 摘要

这组来源覆盖线程与进程、多线程基础、死锁、JMM、volatile、锁、ThreadLocal、线程池、Future、AQS、虚拟线程等内容，是 Java 并发复习的主干资料。

## 来源文件

- [Java并发常见面试题总结（上）](Clippings/Java并发常见面试题总结（上）.md) - 线程、多线程、死锁
- [Java并发常见面试题总结（中）](Clippings/Java并发常见面试题总结（中）.md) - JMM、volatile、乐观锁/悲观锁、synchronized、ReentrantLock
- [Java并发常见面试题总结（下）](Clippings/Java并发常见面试题总结（下）.md) - ThreadLocal、线程池、Future、AQS、虚拟线程
- [多线程相关面试题](Clippings/多线程相关面试题.md) - 多线程高级进阶与高频实战问答

## 关键信息

- 前三篇大而全地梳理了并发脉络，新进的 `多线程相关面试题` 集中于高频实战场景的深入剖析（例如线程池执行流程、拒绝策略选择、ThreadLocal 内存泄漏成因及 AQS 底层状态变化），并与操作系统中的进程线程模型、JVM 类加载相呼应。
- 内容与操作系统、JVM、Spring `@Async` 主题直接相关。
- 适合作为并发知识主线，后续可以拆出锁、线程池、AQS 等专题页。

## 相关实体

- Java
- [itheima](wiki/entities/itheima.md)
- JavaGuide

## 相关概念

- 线程
- JMM
- volatile
- 锁
- 线程池
- AQS
- 虚拟线程

## 待确认问题

- 是否要单独沉淀一页“并发问题排查”主题，把死锁、可见性、竞态、线程池配置合并起来。

## 关联知识页

- [Java并发](wiki/topics/Java并发.md)
