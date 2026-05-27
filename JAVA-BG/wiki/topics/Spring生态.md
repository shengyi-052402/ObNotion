---
type: wiki
updated: 2026-04-24
tags:
  - Spring
  - Java
---

# Spring生态

当前仓库中的 Spring 资料已经覆盖全景问答和若干核心机制专题，可以作为后端框架主线来维护。

## 正文

- 基础主线原理与高频实战解析：
  - **Bean 单例线程安全**：Spring 中的 Bean 默认是单例（Singleton）的，其本身**不是线程安全的**。如果单例 Bean 中含有可变的成员变量状态（例如非只读的实例域），多线程并发访问时需要自行同步或将 Bean 作用域（@Scope）声明为“prototype”原型。通常只读的 Service 或 Dao 是无状态的，天然线程安全。
  - **Bean 的生命周期**：
    1. **解析定义**：通过 `BeanDefinition` 加载类的属性定义。
    2. **实例化**：调用构造函数创建对象。
    3. **属性赋值**：执行依赖注入（Set注入或 @Autowired 注解注入）。
    4. **Aware 接口回调**：处理 BeanNameAware、BeanFactoryAware 等。
    5. **前置增强**：执行 `BeanPostProcessor` 的前置初始化方法。
    6. **初始化**：执行 @PostConstruct、InitializingBean 接口方法或自定义 `init-method`。
    7. **后置增强**：执行 `BeanPostProcessor` 的后置初始化方法（**AOP 代理对象在此步生成**）。
    8. **销毁**：执行销毁回调。
  - **三级缓存与循环依赖**：
    - 循环依赖指 A 依赖 B，B 依赖 A，最终形成依赖闭环。Spring 利用三级缓存解决了单例 Bean 属性注入时的循环依赖问题：
      - 一级缓存 `singletonObjects`：存放完全初始化好的单例 Bean。
      - 二级缓存 `earlySingletonObjects`：存放提前暴露的、生命周期未走完的早期半成品 Bean。
      - 三级缓存 `singletonFactories`：存放创建早期半成品对象的工厂（`ObjectFactory<?>`）。
    - **解决流程**：创建 A 对象 -> 实例化后放入三级缓存工厂 -> A 发现依赖 B -> 触发 B 的创建流程 -> B 实例化并放入三级缓存 -> B 发现依赖 A -> B 从三级缓存获取 A 的 ObjectFactory，生成 A 的早期引用（若是 AOP 则是早期代理对象）存入二级缓存并清除三级缓存 -> B 注入 A 并完成初始化，存入一级缓存 -> 回到 A 的初始化，注入 B -> A 完成初始化，清除二级缓存，存入一级缓存。*注意：构造方法注入造成的循环依赖无法由三级缓存解决，需配合 `@Lazy` 延迟加载。*
  - **声明式事务失效场景**：
    1. **异常被捕获而未抛出**：Spring 基于 AOP 环绕通知拦截异常以触发回滚，若业务代码捕获异常而没有重新往外抛，事务将不会回滚。
    2. **抛出了受检（编译时）异常**：Spring 默认只在遇到运行时异常（RuntimeException）或 Error 时回滚。解决方案：配置 `@Transactional(rollbackFor = Exception.class)`。
    3. **非 public 方法**：Spring 事务代理默认仅能增强 public 修饰的方法，其他权限修饰符将使事务增强失效。
  - **Spring Boot 自动装配原理**：
    - 核心是由主导引导类上的 `@SpringBootApplication` 中封装的三个注解驱动：`@SpringBootConfiguration`、`@ComponentScan` 以及核心的 `@EnableAutoConfiguration`。
    - `@EnableAutoConfiguration` 通过 `@Import` 引入自动配置选择器，读取 classpath 下所有 jar 包中 `META-INF/spring.factories` 文件里配置的 AutoConfiguration 全限定类名。
    - 读取后根据配置类上的条件注解（如 `@ConditionalOnClass` 是否包含对应的字节码类，`@ConditionalOnMissingBean` 容器中是否没有此 Bean）决定是否激活并加载该配置，将 Bean 注册进 Spring 容器中，实现开箱即用。
- 机制专题还包括常用注解、设计模式和 `@Async` 异步处理。
- 这组资料最适合拆成“概念理解 + 原理机制 + 工程使用建议”三层来持续扩写。

## 相关链接

- [Java基础](wiki/topics/Java基础.md)
- [Java并发](wiki/topics/Java并发.md)
- [MyBatis](wiki/topics/MyBatis.md)
- [MySQL](wiki/topics/MySQL.md)

## 来源

- [Spring来源整理](raw/sources/Spring来源整理.md)
