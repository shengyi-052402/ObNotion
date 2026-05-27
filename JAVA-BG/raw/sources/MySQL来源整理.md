---
type: source
status: processed
source_type: clippings
created: 2026-04-24
tags:
  - MySQL
  - 数据库
---

# MySQL来源整理

## 摘要

这组来源覆盖 MySQL 基础、索引、事务、锁、日志、存储引擎以及性能优化规范，既有问答型材料，也有偏实践规范的总结。

## 来源文件

- [MySQL常见面试题总结](Clippings/MySQL常见面试题总结.md) - 基础、字段类型、架构、存储引擎、索引
- [MySQL面试题](Clippings/MySQL面试题.md) - SQL 基础、事务、锁、日志等更完整的问题集
- [MySQL高性能优化规范建议总结](Clippings/MySQL高性能优化规范建议总结.md) - 命名、建模、字段、索引、SQL、操作规范
- [MySQL面试题-参考回答](Clippings/MySQL面试题-参考回答.md) - 实战高频问答与底层机制（MVCC、隔离级别、锁）补充

## 关键信息

- 面试题与高性能规范共同奠定基础理论，新入库的 `MySQL面试题-参考回答` 聚焦于核心实战场景问答，如 MVCC 底层（隐藏字段、undo log、ReadView 判定逻辑）、快照读与当前读的区别、行锁/表锁/意向锁及分库分表实践。
- “高性能优化规范”更偏日常开发约束，可单独作为实践清单使用。

## 相关实体

- MySQL
- [itheima](wiki/entities/itheima.md)
- JavaGuide
- 小林 coding

## 相关概念

- 存储引擎
- 索引
- 事务
- 锁
- 日志
- SQL 优化

## 待确认问题

- 后续是否需要按“索引”“事务与锁”“日志”拆成更细的主题页。

## 关联知识页

- [MySQL](wiki/topics/MySQL.md)
