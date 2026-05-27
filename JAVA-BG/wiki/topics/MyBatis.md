---
type: wiki
updated: 2026-04-24
tags:
  - MyBatis
  - 持久层
---

# MyBatis

当前 MyBatis 资料还比较薄，主要作为持久层框架主题的入口页存在，后续应当继续补充执行流程和缓存机制相关资料。

## 正文

- 核心运行原理与高级特性深挖：
  - **MyBatis 执行流程**：
    1. **读取配置**：加载全局配置文件 `mybatis-config.xml` 以及 Mapper 映射文件，解析运行环境。
    2. **构建工厂**：根据配置构造 `SqlSessionFactory`（会话工厂，生命周期与应用一致，通常单例且交由 Spring 管理）。
    3. **创建会话**：通过工厂创建 `SqlSession`（代表一次数据库连接会话，内部包含执行 SQL 的所有方法，线程不安全，用完需关闭）。
    4. **执行器调度**：SqlSession 委派 `Executor` 执行器处理具体的数据库交互，Executor 负责二级缓存的维护。
    5. **MappedStatement解析**：Executor 在执行方法时，读取对应的 `MappedStatement` 对象（封装了 Mapper.xml 中的一条 SQL 节点、参数映射及结果集映射信息）。
    6. **参数与结果集映射**：Executor 通过 `StatementHandler` 执行 SQL，完成输入参数的映射绑定以及输出结果的 ORM 封装映射。
  - **延迟加载（Lazy Loading）底理解**：
    - 延迟加载指在需要使用关联对象的数据时才去执行 SQL 加载，不使用则不加载（支持一对一 `association` 和一对多 `collection`）。
    - **底层原理**：主要依赖 **CGLIB 动态代理**。当开启延迟加载（配置 `lazyLoadingEnabled=true`）后，MyBatis 不会直接装配关联对象，而是使用 CGLIB 为目标实体生成一个代理对象。当调用关联对象的 Getter 方法时，会被代理对象的拦截器（MethodInterceptor）拦截，拦截器发现该属性值为 null，则会动态发送预先配置好的 SQL 语句去查询数据库，最后调用 Setter 方法回写属性，完成透明的按需加载。
  - **一级与二级缓存机制**：
    - **一级缓存**：基于 `PerpetualCache` 的 HashMap 本地缓存，其**作用域为 SqlSession**。在同一个 SqlSession 内执行相同查询时会直接命中缓存。当 Session 提交（Commit）、关闭（Close）或执行了增删改（insert/update/delete）操作，该 Session 下的一级缓存会被清空。默认开启。
    - **二级缓存**：基于 **Namespace（Mapper命名空间）** 起作用，跨 SqlSession 共享。需要显式开启（全局配置 `cacheEnabled=true` 并在具体 Mapper.xml 中加入 `<cache />` 标签）。同样在 Namespace 内发生增删改操作后，该空间下所有 select 的二级缓存会被 clear 清空。
    - *注意：二级缓存由于以 Namespace 分割，在多表联合查询且分布在不同 Namespace 时，极易发生数据脏读。多表级联频繁写操作时建议谨慎开启或配合第三方 Redis 缓存管理。*
- 这个主题与 Spring 容器管理以及 MySQL 数据访问实践关系最紧密。

## 相关链接

- [Spring生态](wiki/topics/Spring生态.md)
- [MySQL](wiki/topics/MySQL.md)

## 来源

- [MyBatis来源整理](raw/sources/MyBatis来源整理.md)
