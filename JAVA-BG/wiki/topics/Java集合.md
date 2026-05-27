---
type: wiki
updated: 2026-04-24
tags:
  - Java
  - 集合
---

# Java集合

当前集合资料同时覆盖了集合框架结构和实际开发中的使用注意事项，适合一边掌握原理，一边形成编码习惯。

## 正文

- 结构层面，重点是 List、Set、Queue、Map 四类集合的特点与差异。
- 实践层面，重点是判空、遍历删除（避免使用 foreach，应采用 Iterator / removeIf 以免触发 fail-fast 异常）、去重、数组与集合转换、`Collectors.toMap()` 键值对冲突及 null 溢出等高频坑点。
- 底层机制深挖：
  - **ArrayList 扩容**：默认初始容量为 10，当容量不足时触发扩容，每次扩容为原来的 **1.5 倍**（`newCapacity = oldCapacity + (oldCapacity >> 1)`），涉及底层数组拷贝。
  - **HashMap 底层**：JDK 1.8 引入数组+链表+红黑树。当链表长度大于 **8** 且数组长度大于等于 **64** 时，链表转换为红黑树以优化查询效率（O(log N)）；当红黑树节点数减少至 **6** 时退化为链表。寻址采用二次 Hash + 扰动函数减小碰撞。
  - **ConcurrentHashMap 线程安全**：JDK 1.7 采用 **Segment 分段锁**（ReentrantLock 锁段，并发度受 Segment 数量限制）；JDK 1.8 舍弃分段锁，采用**Node数组 + CAS + synchronized** 保证并发安全，将锁的粒度细化至每个数组桶（Head节点），显著提升并发处理能力。
- 集合主题虽然属于 Java 基础，但与并发问题、缓存数据结构、数据库结果映射都有直接联系。

## 相关链接

- [Java基础](wiki/topics/Java基础.md)
- [Java并发](wiki/topics/Java并发.md)
- [Redis](wiki/topics/Redis.md)

## 来源

- [Java集合来源整理](raw/sources/Java集合来源整理.md)
