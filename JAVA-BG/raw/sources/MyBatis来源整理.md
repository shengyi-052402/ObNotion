---
type: source
status: processed
source_type: clippings
created: 2026-04-24
tags:
  - MyBatis
  - 持久层
---

# MyBatis来源整理

## 摘要

这份来源目前只有一篇 MyBatis 面试题总结，适合作为持久层框架主题的起始材料，但深度和覆盖面暂时有限。

## 来源文件

- [MyBatis常见面试题总结](Clippings/MyBatis常见面试题总结.md) - 基础、#{}与${}区别、XML标签等
- [框架篇面试题-参考回答](Clippings/框架篇面试题-参考回答.md) - MyBatis 核心流程、延迟加载与一级/二级缓存实战问答补充

## 关键信息

- 首篇面试题梳理了基本概念，新增的 `框架篇面试题-参考回答` 重点补足了 MyBatis 的核心执行原理与高级特性：详细阐明了 MyBatis 的核心执行步骤（SqlSessionFactory -> SqlSession -> Executor -> MappedStatement）；深剖了基于 CGLIB 动态代理的延迟加载底层原理；详述了一级缓存与二级缓存的作用域、共享清理时机及并发数据脏读的安全风险。

## 相关实体

- MyBatis
- JavaGuide

## 相关概念

- Mapper
- 一级缓存
- 二级缓存
- 动态 SQL

## 待确认问题

- 后续是否补充执行流程、插件机制、缓存实现等更深入内容。

## 关联知识页

- [MyBatis](wiki/topics/MyBatis.md)
