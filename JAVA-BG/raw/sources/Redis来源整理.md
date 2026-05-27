---
type: source
status: processed
source_type: clippings
created: 2026-04-24
tags:
  - Redis
  - 缓存
---

# Redis来源整理

## 摘要

这份来源围绕 Redis 的数据结构、线程模型、事务、日志、淘汰与过期策略、集群等面试高频问题，适合作为 Redis 主题的基础材料。

## 来源文件

- [Redis面试题](Clippings/Redis面试题.md) - 数据结构、线程模型、事务、日志、淘汰策略、集群
- [Redis面试题-参考回答](Clippings/Redis面试题-参考回答.md) - 生产实战（双写一致性、缓存穿透/击穿/雪崩、分布式锁）

## 关键信息

- 首份面试题确立了底层的理论基石，新摄入的 `Redis面试题-参考回答` 重点补齐了生产应用中极关键的工程场景，包括：缓存穿透/击穿/雪崩的完备方案（布隆过滤器、互斥锁、热点永不过期）、数据双写一致性（双删延迟、延迟MQ、Redisson读写锁）、分布式锁底层实现（Redisson看门狗与红锁）等。

## 相关实体

- Redis
- 小林 coding

## 相关概念

- 数据结构
- 单线程模型
- 持久化
- 淘汰策略
- 集群

## 待确认问题

- 是否要引入更多偏工程场景的 Redis 资料。

## 关联知识页

- [Redis](wiki/topics/Redis.md)
