---
title: "如何基于Redis实现消息队列？"
source: "https://javaguide.cn/database/redis/redis-stream-mq.html"
author:
  - "[[Guide]]"
published: 2026-04-08
created: 2026-04-24
description: "讲解 Redis 做消息队列的三种方式：List、Pub/Sub、Stream。对比生产级 MQ 核心能力，详解 Redis 5.0 Stream 的消费者组、ACK 机制及与 Kafka/RabbitMQ 的适用场景对比。"
tags:
  - "clippings"
---
先说结论： **可以是可以，但要看具体场景。和专业的消息队列（如 Kafka、RabbitMQ）相比，还是有一些欠缺的地方。**

正式开始介绍之前，我们先来看看： **一个生产级 MQ 需要具备哪些核心能力？**

| 能力维度 | 定义 | 关键指标/特征 |
| --- | --- | --- |
| **持久化** | 消息写入后不因进程/节点故障丢失 | 同步刷盘/多副本确认、RPO ≈ 0 |
| **至少一次投递** | 消息最终被消费，允许重复 | 需配合消费者幂等性 |
| **消费确认** | 消费者显式告知处理成功 | ACK 机制、超时重试、死信队列 |
| **消息重试** | 消费失败可自动重新投递 | 退避策略、最大重试次数、死信转移 |
| **消费者组** | 多消费者协作消费，故障自动转移 | 组内负载均衡、分区分配、Rebalance |
| **消息堆积能力** | 生产速率 > 消费速率时的缓冲能力 | 磁盘存储、TTL、堆积告警 |
| **顺序保证** | 消息按发送顺序被消费 | 分区有序/全局有序、乱序惩罚 |
| **可扩展性** | 水平扩展以提升吞吐或容灾 | 分片机制、无状态 Broker、动态扩缩容 |

Redis 提供了多种实现 MQ 的方式，从早期的 `List` 到 `Pub/Sub` ，再到 Redis 5.0 新增的 `Stream` 数据结构（基于有序链表实现，支持消费者组和 ACK 机制，可用于构建轻量级消息队列）。

### 第一阶段：早期用 List 数据结构

**Redis 2.0 之前，如果想要使用 Redis 来做消息队列的话，只能通过 List 来实现。**

通过 `RPUSH/LPOP` 或者 `LPUSH/RPOP` 即可实现简易版消息队列：

```bash
# 生产者生产消息
> RPUSH myList msg1 msg2
(integer) 2
> RPUSH myList msg3
(integer) 3
# 消费者消费消息
> LPOP myList
"msg1"
```

不过，通过 `RPUSH/LPOP` 或者 `LPUSH/RPOP` 这样的方式存在性能问题，我们需要不断轮询去调用 `RPOP` 或 `LPOP` 来消费消息。当 List 为空时，大部分的轮询的请求都是无效请求，这种方式大量浪费了系统资源。

因此，Redis 还提供了 `BLPOP` 、 `BRPOP` 这种阻塞式读取的命令（带 B-Blocking 的都是阻塞式），并且还支持一个超时参数。如果 List 为空，Redis 服务端不会立刻返回结果，它会等待 List 中有新数据后再返回或者是等待最多一个超时时间后返回空。如果将超时时间设置为 0 时，即可无限等待，直到弹出消息

```bash
# 超时时间为 10s
# 如果有数据立刻返回，否则最多等待10秒
> BRPOP myList 10
null
```

List 实现消息队列功能太简单，像消息确认机制等功能还需要我们自己实现。 **最致命的是，它不支持一个消息被多个消费者消费（广播），而且消息一旦被取出，就没有了，如果消费者处理失败，消息就永久丢失了。**

### 第二阶段：引入 Pub/Sub（发布/订阅）模式

**Redis 2.0 引入了发布订阅 (Pub/Sub) 功能，解决了 List 实现消息队列没有广播机制的问题。**

Pub/Sub 中引入了一个概念叫 **Channel（频道）** ，发布订阅机制的实现就是基于这个 Channel 来做的。

Pub/Sub 涉及发布者（Publisher）和订阅者（Subscriber，也叫消费者）两个角色：

- 发布者通过 `PUBLISH` 投递消息给指定 Channel。
- 订阅者通过 `SUBSCRIBE` 订阅它关心的 Channel。并且，订阅者可以订阅一个或者多个 Channel。

也就是说，多个消费者可以订阅同一个 Channel，生产者向这个 Channel 发布消息，所有订阅者都能收到。

我们这里启动 3 个 Redis 客户端来简单演示一下：

![Pub/Sub 实现消息队列演示](https://oss.javaguide.cn/github/javaguide/database/redis/redis-pubsub-message-queue.png)

Pub/Sub 既能单播又能广播，还支持 Channel 的简单正则匹配。

Pub/Sub 有一个致命的缺陷： **它发后即忘，完全没有持久化和可靠性保证** 。 如果消息发布时，某个消费者不在线，或者网络抖动了一下，那这条消息对它来说就永远丢失了。此外，它也 **没有 ACK 机制** ，无法知道消费者是否成功处理，更别提 **消息堆积** 的问题了。所以，Pub/Sub 只适合做一些对可靠性要求极低的实时通知，绝对不能用于任何严肃的业务消息队列。

### 第三阶段：Redis 5.0 新增 Stream

Redis 5.0 新增了 `Stream` 数据结构。这是一个基于 Radix Tree（基数树）实现的有序消息日志，天然支持消费者组和 ACK 机制，可用于构建轻量级消息队列。

**为什么要用 Radix Tree？** 很多人好奇，为什么不继续用 `List/LinkedList` ？

1. **内存极度压缩** ： `Stream` 的消息 ID（如 `1625000000000-0` ）是高度有序且前缀高度重合的。Radix Tree 是一种压缩前缀树，它会将具有相同前缀的节点合并。而 List/LinkedList  
	每个元素都要完整的链表节点开销，并且无法利用 ID 的前缀重复特性来节省空间。
2. **高效检索** ：在处理数百万级消息堆积时，Radix Tree 能保持极高的查询效率，这也是 `Stream` 能支持大数据量范围查询（ `XRANGE` ）的底层底气。相比之下， `List/LinkedList` 只能从头尾操作，无法高效按 ID 范围查询，执行 `XRANGE` 需要遍历整个列表。

它借鉴了 Kafka 等专业 MQ 的核心概念：

1. **消费者组（Consumer Groups）** ：实现消息在多个消费者间的负载均衡，支持故障自动转移。
2. **持久化** ：可以通过 RDB 和 AOF 保证消息在 Redis 重启后不丢失（取决于 `appendfsync` 配置， `everysec` 模式下通常最多丢失 1 秒数据）。
3. **ACK 机制** ：消费者处理完消息后，需要手动 `XACK` 确认，否则消息会保留在 `Pending List` 中。这保证了消息至少被成功消费一次。
4. **消息回溯与转移** ：支持 `XRANGE` 按时间范围回溯消息，以及 `XCLAIM` 将挂起的消息转移到其他消费者处理。

> 🌈 版本演进：
> 
> - Redis 8.2： `XACKDEL` 、 `XDELEX` 、 `XADD` 和 \`XTRIM 命令提供了对流操作如何与多个消费者组交互的细粒度控制，简化了跨不同应用程序的消息处理协调。
> - Redis 8.6：支持幂等消息处理（最多一次生产），防止在使用至少一次交付模式时出现重复条目。此功能可实现可靠的消息提交，并自动去重。

`Stream` 的结构如下：

![](https://oss.javaguide.cn/github/javaguide/database/redis/redis-stream-structure.png)

这是一个有序的消息链表，每个消息都有一个唯一的 ID 和对应的内容。ID 是一个时间戳和序列号的组合，用来保证消息的唯一性和递增性。内容是一个或多个键值对（类似 Hash 基本数据类型），用来存储消息的数据。

这里再对图中涉及到的一些概念，进行简单解释：

- `Consumer Group` ：消费者组用于组织和管理多个消费者。消费者组本身不处理消息，而是再将消息分发给消费者，由消费者进行真正的消费。
- `last_delivered_id` ：标识消费者组当前消费位置的游标，消费者组中任意一个消费者读取了消息都会使 last\_delivered\_id 往前移动。
- `pending_ids` ：记录已经被客户端消费但没有 ack 的消息的 ID。

下面是 `Stream` 用作消息队列时常用的命令：

- `XADD` ：向流中添加新的消息。
- `XREAD` ：从流中读取消息。
- `XREADGROUP` ：从消费组中读取消息。
- `XRANGE` ：根据消息 ID 范围读取流中的消息。
- `XREVRANGE` ：与 `XRANGE` 类似，但以相反顺序返回结果。
- `XDEL` ：从流中删除消息。
- `XTRIM` ：修剪流的长度，可以指定修建策略（ `MAXLEN` / `MINID` ）。
- `XLEN` ：获取流的长度。
- `XGROUP CREATE` ：创建消费者组。
- `XGROUP DESTROY` ：删除消费者组。
- `XGROUP DELCONSUMER` ：从消费者组中删除一个消费者。
- `XGROUP SETID` ：为消费者组设置新的最后递送消息 ID。
- `XACK` ：确认消费组中的消息已被处理。
- `XPENDING` ：查询消费组中挂起（未确认）的消息。
- `XCLAIM` ：将挂起的消息从一个消费者转移到另一个消费者。
- `XINFO` ：获取流（ `XINFO STREAM` ）、消费组（ `XINFO GROUPS` ）或消费者（ `XINFO CONSUMERS` ）的详细信息。

下面这张时序图展示了 Stream 消费者组消息流转与 ACK 机制：

<svg id="v-8" width="1150" xmlns="http://www.w3.org/2000/svg" height="1178" viewBox="-50 -10 1150 1178" role="graphics-document document" aria-roledescription="sequence"><g><rect x="900" y="1092" fill="#eaeaea" stroke="#666" width="150" height="65" name="C2" rx="3" ry="3"></rect><text x="975" y="1124.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="975" dy="0">Consumer-2</tspan></text></g> <g><rect x="700" y="1092" fill="#eaeaea" stroke="#666" width="150" height="65" name="C1" rx="3" ry="3"></rect><text x="775" y="1124.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="775" dy="0">Consumer-1</tspan></text></g> <g><rect x="495" y="1092" fill="#eaeaea" stroke="#666" width="150" height="65" name="CG" rx="3" ry="3"></rect><text x="570" y="1124.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="570" dy="-8">Consumer Group</tspan></text> <text x="570" y="1124.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="570" dy="8">(group_a)</tspan></text></g> <g><rect x="290" y="1092" fill="#eaeaea" stroke="#666" width="150" height="65" name="R" rx="3" ry="3"></rect><text x="365" y="1124.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="365" dy="-8">Redis Stream</tspan></text> <text x="365" y="1124.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="365" dy="8">(my_stream)</tspan></text></g> <g><rect x="0" y="1092" fill="#eaeaea" stroke="#666" width="150" height="65" name="P" rx="3" ry="3"></rect><text x="75" y="1124.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="75" dy="0">Producer</tspan></text></g> <g><line id="actor6" x1="975" y1="65" x2="975" y2="1092" stroke-width="0.5px" stroke="#999" name="C2"></line><g id="root-6"><rect x="900" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" name="C2" rx="3" ry="3"></rect><text x="975" y="32.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="975" dy="0">Consumer-2</tspan></text></g></g> <g><line id="actor5" x1="775" y1="65" x2="775" y2="1092" stroke-width="0.5px" stroke="#999" name="C1"></line><g id="root-5"><rect x="700" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" name="C1" rx="3" ry="3"></rect><text x="775" y="32.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="775" dy="0">Consumer-1</tspan></text></g></g> <g><line id="actor4" x1="570" y1="65" x2="570" y2="1092" stroke-width="0.5px" stroke="#999" name="CG"></line><g id="root-4"><rect x="495" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" name="CG" rx="3" ry="3"></rect><text x="570" y="32.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="570" dy="-8">Consumer Group</tspan></text> <text x="570" y="32.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="570" dy="8">(group_a)</tspan></text></g></g> <g><line id="actor3" x1="365" y1="65" x2="365" y2="1092" stroke-width="0.5px" stroke="#999" name="R"></line><g id="root-3"><rect x="290" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" name="R" rx="3" ry="3"></rect><text x="365" y="32.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="365" dy="-8">Redis Stream</tspan></text> <text x="365" y="32.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="365" dy="8">(my_stream)</tspan></text></g></g> <g><line id="actor2" x1="75" y1="65" x2="75" y2="1092" stroke-width="0.5px" stroke="#999" name="P"></line><g id="root-2"><rect x="0" y="0" fill="#eaeaea" stroke="#666" width="150" height="65" name="P" rx="3" ry="3"></rect><text x="75" y="32.5" dominant-baseline="central" alignment-baseline="central" style="text-anchor: middle; font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="75" dy="0">Producer</tspan></text></g></g> <g></g><defs><symbol id="computer" width="24" height="24"><path transform="scale(.5)" d="M2 2v13h20v-13h-20zm18 11h-16v-9h16v9zm-10.228 6l.466-1h3.524l.467 1h-4.457zm14.228 3h-24l2-6h2.104l-1.33 4h18.45l-1.297-4h2.073l2 6zm-5-10h-14v-7h14v7z"></path></symbol></defs><defs><symbol id="database" fill-rule="evenodd" clip-rule="evenodd"><path transform="scale(.5)" d="M12.258.001l.256.004.255.005.253.008.251.01.249.012.247.015.246.016.242.019.241.02.239.023.236.024.233.027.231.028.229.031.225.032.223.034.22.036.217.038.214.04.211.041.208.043.205.045.201.046.198.048.194.05.191.051.187.053.183.054.18.056.175.057.172.059.168.06.163.061.16.063.155.064.15.066.074.033.073.033.071.034.07.034.069.035.068.035.067.035.066.035.064.036.064.036.062.036.06.036.06.037.058.037.058.037.055.038.055.038.053.038.052.038.051.039.05.039.048.039.047.039.045.04.044.04.043.04.041.04.04.041.039.041.037.041.036.041.034.041.033.042.032.042.03.042.029.042.027.042.026.043.024.043.023.043.021.043.02.043.018.044.017.043.015.044.013.044.012.044.011.045.009.044.007.045.006.045.004.045.002.045.001.045v17l-.001.045-.002.045-.004.045-.006.045-.007.045-.009.044-.011.045-.012.044-.013.044-.015.044-.017.043-.018.044-.02.043-.021.043-.023.043-.024.043-.026.043-.027.042-.029.042-.03.042-.032.042-.033.042-.034.041-.036.041-.037.041-.039.041-.04.041-.041.04-.043.04-.044.04-.045.04-.047.039-.048.039-.05.039-.051.039-.052.038-.053.038-.055.038-.055.038-.058.037-.058.037-.06.037-.06.036-.062.036-.064.036-.064.036-.066.035-.067.035-.068.035-.069.035-.07.034-.071.034-.073.033-.074.033-.15.066-.155.064-.16.063-.163.061-.168.06-.172.059-.175.057-.18.056-.183.054-.187.053-.191.051-.194.05-.198.048-.201.046-.205.045-.208.043-.211.041-.214.04-.217.038-.22.036-.223.034-.225.032-.229.031-.231.028-.233.027-.236.024-.239.023-.241.02-.242.019-.246.016-.247.015-.249.012-.251.01-.253.008-.255.005-.256.004-.258.001-.258-.001-.256-.004-.255-.005-.253-.008-.251-.01-.249-.012-.247-.015-.245-.016-.243-.019-.241-.02-.238-.023-.236-.024-.234-.027-.231-.028-.228-.031-.226-.032-.223-.034-.22-.036-.217-.038-.214-.04-.211-.041-.208-.043-.204-.045-.201-.046-.198-.048-.195-.05-.19-.051-.187-.053-.184-.054-.179-.056-.176-.057-.172-.059-.167-.06-.164-.061-.159-.063-.155-.064-.151-.066-.074-.033-.072-.033-.072-.034-.07-.034-.069-.035-.068-.035-.067-.035-.066-.035-.064-.036-.063-.036-.062-.036-.061-.036-.06-.037-.058-.037-.057-.037-.056-.038-.055-.038-.053-.038-.052-.038-.051-.039-.049-.039-.049-.039-.046-.039-.046-.04-.044-.04-.043-.04-.041-.04-.04-.041-.039-.041-.037-.041-.036-.041-.034-.041-.033-.042-.032-.042-.03-.042-.029-.042-.027-.042-.026-.043-.024-.043-.023-.043-.021-.043-.02-.043-.018-.044-.017-.043-.015-.044-.013-.044-.012-.044-.011-.045-.009-.044-.007-.045-.006-.045-.004-.045-.002-.045-.001-.045v-17l.001-.045.002-.045.004-.045.006-.045.007-.045.009-.044.011-.045.012-.044.013-.044.015-.044.017-.043.018-.044.02-.043.021-.043.023-.043.024-.043.026-.043.027-.042.029-.042.03-.042.032-.042.033-.042.034-.041.036-.041.037-.041.039-.041.04-.041.041-.04.043-.04.044-.04.046-.04.046-.039.049-.039.049-.039.051-.039.052-.038.053-.038.055-.038.056-.038.057-.037.058-.037.06-.037.061-.036.062-.036.063-.036.064-.036.066-.035.067-.035.068-.035.069-.035.07-.034.072-.034.072-.033.074-.033.151-.066.155-.064.159-.063.164-.061.167-.06.172-.059.176-.057.179-.056.184-.054.187-.053.19-.051.195-.05.198-.048.201-.046.204-.045.208-.043.211-.041.214-.04.217-.038.22-.036.223-.034.226-.032.228-.031.231-.028.234-.027.236-.024.238-.023.241-.02.243-.019.245-.016.247-.015.249-.012.251-.01.253-.008.255-.005.256-.004.258-.001.258.001zm-9.258 20.499v.01l.001.021.003.021.004.022.005.021.006.022.007.022.009.023.01.022.011.023.012.023.013.023.015.023.016.024.017.023.018.024.019.024.021.024.022.025.023.024.024.025.052.049.056.05.061.051.066.051.07.051.075.051.079.052.084.052.088.052.092.052.097.052.102.051.105.052.11.052.114.051.119.051.123.051.127.05.131.05.135.05.139.048.144.049.147.047.152.047.155.047.16.045.163.045.167.043.171.043.176.041.178.041.183.039.187.039.19.037.194.035.197.035.202.033.204.031.209.03.212.029.216.027.219.025.222.024.226.021.23.02.233.018.236.016.24.015.243.012.246.01.249.008.253.005.256.004.259.001.26-.001.257-.004.254-.005.25-.008.247-.011.244-.012.241-.014.237-.016.233-.018.231-.021.226-.021.224-.024.22-.026.216-.027.212-.028.21-.031.205-.031.202-.034.198-.034.194-.036.191-.037.187-.039.183-.04.179-.04.175-.042.172-.043.168-.044.163-.045.16-.046.155-.046.152-.047.148-.048.143-.049.139-.049.136-.05.131-.05.126-.05.123-.051.118-.052.114-.051.11-.052.106-.052.101-.052.096-.052.092-.052.088-.053.083-.051.079-.052.074-.052.07-.051.065-.051.06-.051.056-.05.051-.05.023-.024.023-.025.021-.024.02-.024.019-.024.018-.024.017-.024.015-.023.014-.024.013-.023.012-.023.01-.023.01-.022.008-.022.006-.022.006-.022.004-.022.004-.021.001-.021.001-.021v-4.127l-.077.055-.08.053-.083.054-.085.053-.087.052-.09.052-.093.051-.095.05-.097.05-.1.049-.102.049-.105.048-.106.047-.109.047-.111.046-.114.045-.115.045-.118.044-.12.043-.122.042-.124.042-.126.041-.128.04-.13.04-.132.038-.134.038-.135.037-.138.037-.139.035-.142.035-.143.034-.144.033-.147.032-.148.031-.15.03-.151.03-.153.029-.154.027-.156.027-.158.026-.159.025-.161.024-.162.023-.163.022-.165.021-.166.02-.167.019-.169.018-.169.017-.171.016-.173.015-.173.014-.175.013-.175.012-.177.011-.178.01-.179.008-.179.008-.181.006-.182.005-.182.004-.184.003-.184.002h-.37l-.184-.002-.184-.003-.182-.004-.182-.005-.181-.006-.179-.008-.179-.008-.178-.01-.176-.011-.176-.012-.175-.013-.173-.014-.172-.015-.171-.016-.17-.017-.169-.018-.167-.019-.166-.02-.165-.021-.163-.022-.162-.023-.161-.024-.159-.025-.157-.026-.156-.027-.155-.027-.153-.029-.151-.03-.15-.03-.148-.031-.146-.032-.145-.033-.143-.034-.141-.035-.14-.035-.137-.037-.136-.037-.134-.038-.132-.038-.13-.04-.128-.04-.126-.041-.124-.042-.122-.042-.12-.044-.117-.043-.116-.045-.113-.045-.112-.046-.109-.047-.106-.047-.105-.048-.102-.049-.1-.049-.097-.05-.095-.05-.093-.052-.09-.051-.087-.052-.085-.053-.083-.054-.08-.054-.077-.054v4.127zm0-5.654v.011l.001.021.003.021.004.021.005.022.006.022.007.022.009.022.01.022.011.023.012.023.013.023.015.024.016.023.017.024.018.024.019.024.021.024.022.024.023.025.024.024.052.05.056.05.061.05.066.051.07.051.075.052.079.051.084.052.088.052.092.052.097.052.102.052.105.052.11.051.114.051.119.052.123.05.127.051.131.05.135.049.139.049.144.048.147.048.152.047.155.046.16.045.163.045.167.044.171.042.176.042.178.04.183.04.187.038.19.037.194.036.197.034.202.033.204.032.209.03.212.028.216.027.219.025.222.024.226.022.23.02.233.018.236.016.24.014.243.012.246.01.249.008.253.006.256.003.259.001.26-.001.257-.003.254-.006.25-.008.247-.01.244-.012.241-.015.237-.016.233-.018.231-.02.226-.022.224-.024.22-.025.216-.027.212-.029.21-.03.205-.032.202-.033.198-.035.194-.036.191-.037.187-.039.183-.039.179-.041.175-.042.172-.043.168-.044.163-.045.16-.045.155-.047.152-.047.148-.048.143-.048.139-.05.136-.049.131-.05.126-.051.123-.051.118-.051.114-.052.11-.052.106-.052.101-.052.096-.052.092-.052.088-.052.083-.052.079-.052.074-.051.07-.052.065-.051.06-.05.056-.051.051-.049.023-.025.023-.024.021-.025.02-.024.019-.024.018-.024.017-.024.015-.023.014-.023.013-.024.012-.022.01-.023.01-.023.008-.022.006-.022.006-.022.004-.021.004-.022.001-.021.001-.021v-4.139l-.077.054-.08.054-.083.054-.085.052-.087.053-.09.051-.093.051-.095.051-.097.05-.1.049-.102.049-.105.048-.106.047-.109.047-.111.046-.114.045-.115.044-.118.044-.12.044-.122.042-.124.042-.126.041-.128.04-.13.039-.132.039-.134.038-.135.037-.138.036-.139.036-.142.035-.143.033-.144.033-.147.033-.148.031-.15.03-.151.03-.153.028-.154.028-.156.027-.158.026-.159.025-.161.024-.162.023-.163.022-.165.021-.166.02-.167.019-.169.018-.169.017-.171.016-.173.015-.173.014-.175.013-.175.012-.177.011-.178.009-.179.009-.179.007-.181.007-.182.005-.182.004-.184.003-.184.002h-.37l-.184-.002-.184-.003-.182-.004-.182-.005-.181-.007-.179-.007-.179-.009-.178-.009-.176-.011-.176-.012-.175-.013-.173-.014-.172-.015-.171-.016-.17-.017-.169-.018-.167-.019-.166-.02-.165-.021-.163-.022-.162-.023-.161-.024-.159-.025-.157-.026-.156-.027-.155-.028-.153-.028-.151-.03-.15-.03-.148-.031-.146-.033-.145-.033-.143-.033-.141-.035-.14-.036-.137-.036-.136-.037-.134-.038-.132-.039-.13-.039-.128-.04-.126-.041-.124-.042-.122-.043-.12-.043-.117-.044-.116-.044-.113-.046-.112-.046-.109-.046-.106-.047-.105-.048-.102-.049-.1-.049-.097-.05-.095-.051-.093-.051-.09-.051-.087-.053-.085-.052-.083-.054-.08-.054-.077-.054v4.139zm0-5.666v.011l.001.02.003.022.004.021.005.022.006.021.007.022.009.023.01.022.011.023.012.023.013.023.015.023.016.024.017.024.018.023.019.024.021.025.022.024.023.024.024.025.052.05.056.05.061.05.066.051.07.051.075.052.079.051.084.052.088.052.092.052.097.052.102.052.105.051.11.052.114.051.119.051.123.051.127.05.131.05.135.05.139.049.144.048.147.048.152.047.155.046.16.045.163.045.167.043.171.043.176.042.178.04.183.04.187.038.19.037.194.036.197.034.202.033.204.032.209.03.212.028.216.027.219.025.222.024.226.021.23.02.233.018.236.017.24.014.243.012.246.01.249.008.253.006.256.003.259.001.26-.001.257-.003.254-.006.25-.008.247-.01.244-.013.241-.014.237-.016.233-.018.231-.02.226-.022.224-.024.22-.025.216-.027.212-.029.21-.03.205-.032.202-.033.198-.035.194-.036.191-.037.187-.039.183-.039.179-.041.175-.042.172-.043.168-.044.163-.045.16-.045.155-.047.152-.047.148-.048.143-.049.139-.049.136-.049.131-.051.126-.05.123-.051.118-.052.114-.051.11-.052.106-.052.101-.052.096-.052.092-.052.088-.052.083-.052.079-.052.074-.052.07-.051.065-.051.06-.051.056-.05.051-.049.023-.025.023-.025.021-.024.02-.024.019-.024.018-.024.017-.024.015-.023.014-.024.013-.023.012-.023.01-.022.01-.023.008-.022.006-.022.006-.022.004-.022.004-.021.001-.021.001-.021v-4.153l-.077.054-.08.054-.083.053-.085.053-.087.053-.09.051-.093.051-.095.051-.097.05-.1.049-.102.048-.105.048-.106.048-.109.046-.111.046-.114.046-.115.044-.118.044-.12.043-.122.043-.124.042-.126.041-.128.04-.13.039-.132.039-.134.038-.135.037-.138.036-.139.036-.142.034-.143.034-.144.033-.147.032-.148.032-.15.03-.151.03-.153.028-.154.028-.156.027-.158.026-.159.024-.161.024-.162.023-.163.023-.165.021-.166.02-.167.019-.169.018-.169.017-.171.016-.173.015-.173.014-.175.013-.175.012-.177.01-.178.01-.179.009-.179.007-.181.006-.182.006-.182.004-.184.003-.184.001-.185.001-.185-.001-.184-.001-.184-.003-.182-.004-.182-.006-.181-.006-.179-.007-.179-.009-.178-.01-.176-.01-.176-.012-.175-.013-.173-.014-.172-.015-.171-.016-.17-.017-.169-.018-.167-.019-.166-.02-.165-.021-.163-.023-.162-.023-.161-.024-.159-.024-.157-.026-.156-.027-.155-.028-.153-.028-.151-.03-.15-.03-.148-.032-.146-.032-.145-.033-.143-.034-.141-.034-.14-.036-.137-.036-.136-.037-.134-.038-.132-.039-.13-.039-.128-.041-.126-.041-.124-.041-.122-.043-.12-.043-.117-.044-.116-.044-.113-.046-.112-.046-.109-.046-.106-.048-.105-.048-.102-.048-.1-.05-.097-.049-.095-.051-.093-.051-.09-.052-.087-.052-.085-.053-.083-.053-.08-.054-.077-.054v4.153zm8.74-8.179l-.257.004-.254.005-.25.008-.247.011-.244.012-.241.014-.237.016-.233.018-.231.021-.226.022-.224.023-.22.026-.216.027-.212.028-.21.031-.205.032-.202.033-.198.034-.194.036-.191.038-.187.038-.183.04-.179.041-.175.042-.172.043-.168.043-.163.045-.16.046-.155.046-.152.048-.148.048-.143.048-.139.049-.136.05-.131.05-.126.051-.123.051-.118.051-.114.052-.11.052-.106.052-.101.052-.096.052-.092.052-.088.052-.083.052-.079.052-.074.051-.07.052-.065.051-.06.05-.056.05-.051.05-.023.025-.023.024-.021.024-.02.025-.019.024-.018.024-.017.023-.015.024-.014.023-.013.023-.012.023-.01.023-.01.022-.008.022-.006.023-.006.021-.004.022-.004.021-.001.021-.001.021.001.021.001.021.004.021.004.022.006.021.006.023.008.022.01.022.01.023.012.023.013.023.014.023.015.024.017.023.018.024.019.024.02.025.021.024.023.024.023.025.051.05.056.05.06.05.065.051.07.052.074.051.079.052.083.052.088.052.092.052.096.052.101.052.106.052.11.052.114.052.118.051.123.051.126.051.131.05.136.05.139.049.143.048.148.048.152.048.155.046.16.046.163.045.168.043.172.043.175.042.179.041.183.04.187.038.191.038.194.036.198.034.202.033.205.032.21.031.212.028.216.027.22.026.224.023.226.022.231.021.233.018.237.016.241.014.244.012.247.011.25.008.254.005.257.004.26.001.26-.001.257-.004.254-.005.25-.008.247-.011.244-.012.241-.014.237-.016.233-.018.231-.021.226-.022.224-.023.22-.026.216-.027.212-.028.21-.031.205-.032.202-.033.198-.034.194-.036.191-.038.187-.038.183-.04.179-.041.175-.042.172-.043.168-.043.163-.045.16-.046.155-.046.152-.048.148-.048.143-.048.139-.049.136-.05.131-.05.126-.051.123-.051.118-.051.114-.052.11-.052.106-.052.101-.052.096-.052.092-.052.088-.052.083-.052.079-.052.074-.051.07-.052.065-.051.06-.05.056-.05.051-.05.023-.025.023-.024.021-.024.02-.025.019-.024.018-.024.017-.023.015-.024.014-.023.013-.023.012-.023.01-.023.01-.022.008-.022.006-.023.006-.021.004-.022.004-.021.001-.021.001-.021-.001-.021-.001-.021-.004-.021-.004-.022-.006-.021-.006-.023-.008-.022-.01-.022-.01-.023-.012-.023-.013-.023-.014-.023-.015-.024-.017-.023-.018-.024-.019-.024-.02-.025-.021-.024-.023-.024-.023-.025-.051-.05-.056-.05-.06-.05-.065-.051-.07-.052-.074-.051-.079-.052-.083-.052-.088-.052-.092-.052-.096-.052-.101-.052-.106-.052-.11-.052-.114-.052-.118-.051-.123-.051-.126-.051-.131-.05-.136-.05-.139-.049-.143-.048-.148-.048-.152-.048-.155-.046-.16-.046-.163-.045-.168-.043-.172-.043-.175-.042-.179-.041-.183-.04-.187-.038-.191-.038-.194-.036-.198-.034-.202-.033-.205-.032-.21-.031-.212-.028-.216-.027-.22-.026-.224-.023-.226-.022-.231-.021-.233-.018-.237-.016-.241-.014-.244-.012-.247-.011-.25-.008-.254-.005-.257-.004-.26-.001-.26.001z"></path></symbol></defs><defs><symbol id="clock" width="24" height="24"><path transform="scale(.5)" d="M12 2c5.514 0 10 4.486 10 10s-4.486 10-10 10-10-4.486-10-10 4.486-10 10-10zm0-2c-6.627 0-12 5.373-12 12s5.373 12 12 12 12-5.373 12-12-5.373-12-12-12zm5.848 12.459c.202.038.202.333.001.372-1.907.361-6.045 1.111-6.547 1.111-.719 0-1.301-.582-1.301-1.301 0-.512.77-5.447 1.125-7.445.034-.192.312-.181.343.014l.985 6.238 5.394 1.011z"></path></symbol></defs><defs><marker id="arrowhead" refX="7.9" refY="5" markerUnits="userSpaceOnUse" markerWidth="12" markerHeight="12" orient="auto-start-reverse"><path d="M -1 0 L 10 5 L 0 10 z"></path></marker></defs><defs><marker id="crosshead" markerWidth="15" markerHeight="8" orient="auto" refX="4" refY="4.5"><path fill="none" stroke="#000000" stroke-width="1pt" d="M 1,2 L 6,7 M 6,2 L 1,7" style="stroke-dasharray: 0, 0;"></path></marker></defs><defs><marker id="filled-head" refX="15.5" refY="7" markerWidth="20" markerHeight="28" orient="auto"><path d="M 18,7 L9,13 L14,7 L9,1 Z"></path></marker></defs><defs><marker id="sequencenumber" refX="15" refY="15" markerWidth="60" markerHeight="40" orient="auto"><circle cx="15" cy="15" r="6"></circle></marker></defs><g><rect x="437.5" y="287" fill="#EDF2AE" stroke="#666" width="265" height="38"></rect><text x="570" y="292" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="570">1️⃣ last_delivered_id 推进到 1001</tspan></text></g> <g><rect x="415" y="335" fill="#EDF2AE" stroke="#666" width="310" height="38"></rect><text x="570" y="340" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="570">2️⃣ 1001 进入 PEL (Pending Entries List)</tspan></text></g> <g><rect x="495" y="520" fill="#EDF2AE" stroke="#666" width="150" height="38"></rect><text x="570" y="525" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="570">1001 从 PEL 移除</tspan></text></g> <g><rect x="699" y="613" fill="#EDF2AE" stroke="#666" width="152" height="38"></rect><text x="775" y="618" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="775">未 ACK，连接断开</tspan></text></g> <g><rect x="493.5" y="661" fill="#EDF2AE" stroke="#666" width="153" height="57"></rect><text x="570" y="666" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="570">1001 仍在 PEL 中</tspan></text> <text x="570" y="684" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="570">idle time 持续增长</tspan></text></g> <g><rect x="470.5" y="932" fill="#EDF2AE" stroke="#666" width="199" height="38"></rect><text x="570" y="937" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="570">1001 转移到 consumer-2</tspan></text></g> <g><line x1="354" y1="383" x2="986" y2="383" stroke="currentColor" stroke-opacity="0.2"></line><line x1="986" y1="383" x2="986" y2="1072" stroke="currentColor" stroke-opacity="0.2"></line><line x1="354" y1="1072" x2="986" y2="1072" stroke="currentColor" stroke-opacity="0.2"></line><line x1="354" y1="383" x2="354" y2="1072" stroke="currentColor" stroke-opacity="0.2"></line><line x1="354" y1="573" x2="986" y2="573" style="stroke-dasharray: 3, 3;" stroke="currentColor" stroke-opacity="0.2"></line><polygon points="354,383 404,383 404,396 395.6,403 354,403" fill="none" stroke="currentColor"></polygon><text x="379" y="396" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" style="font-size: 16px; font-weight: 400;" fill="currentColor">alt</text> <text x="695" y="401" text-anchor="middle" style="font-size: 16px; font-weight: 400;" fill="currentColor"><tspan x="695">[正常处理完成]</tspan></text> <text x="670" y="591" text-anchor="middle" style="font-size: 16px; font-weight: 400;" fill="currentColor">[消费者崩溃]</text></g> <text x="219" y="80" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">XADD my_stream * field value</text> <line x1="76" y1="111" x2="361" y2="111" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="fill: none;"></line><text x="222" y="126" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">返回 ID = 1001</text> <line x1="364" y1="157" x2="79" y2="157" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line><text x="572" y="172" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">XREADGROUP GROUP group_a consumer-1</text> <text x="572" y="190" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">STREAMS my_stream &gt;</text> <line x1="774" y1="221" x2="369" y2="221" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="fill: none;"></line><text x="569" y="236" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">返回消息 1001</text> <line x1="366" y1="277" x2="771" y2="277" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line><text x="572" y="433" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">XACK my_stream group_a 1001</text> <line x1="774" y1="464" x2="369" y2="464" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="fill: none;"></line><text x="569" y="479" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">OK</text> <line x1="366" y1="510" x2="771" y2="510" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line><text x="672" y="733" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">XPENDING my_stream group_a</text> <line x1="974" y1="764" x2="369" y2="764" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="fill: none;"></line><text x="669" y="779" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">返回 1001 + idle time</text> <line x1="366" y1="820" x2="971" y2="820" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line><text x="672" y="835" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">XCLAIM my_stream group_a consumer-2 60000 1001</text> <line x1="974" y1="866" x2="369" y2="866" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="fill: none;"></line><text x="669" y="881" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">返回 1001</text> <line x1="366" y1="922" x2="971" y2="922" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line><text x="672" y="985" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">XACK my_stream group_a 1001</text> <line x1="974" y1="1016" x2="369" y2="1016" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="fill: none;"></line><text x="669" y="1031" text-anchor="middle" dominant-baseline="middle" alignment-baseline="middle" dy="1em" style="font-size: 16px; font-weight: 400;" fill="currentColor">OK</text><line x1="366" y1="1062" x2="971" y2="1062" stroke-width="2" stroke="none" marker-end="url(#arrowhead)" style="stroke-dasharray: 3, 3; fill: none;"></line></svg>

总的来说， `Stream` 已经可以满足一个消息队列的基本要求了。不过， `Stream` 在实际使用中需要注意以下几点：

1. **持久化限制** ：Redis 5.0 的 Stream 依赖 RDB/AOF 异步持久化，在故障恢复时可能丢失最近未持久化的消息（取决于 `appendfsync` 配置）。AOF 的 `everysec` 模式下通常最多丢失 1 秒数据。
2. **消息堆积受限** ：Redis Stream 的数据存储在内存中，受服务器内存容量限制。相比 Kafka 基于磁盘的存储，Redis Stream 不适合海量堆积场景。
3. **消费组管理** ：Consumer Group 的状态信息（如 `last_delivered_id` ）需要定期维护，长时间未处理的 Pending 消息会占用内存。

下面这张表格是 Redis Stream 和常见 MQ 的对比：

| 维度 | Redis Stream | RabbitMQ | Kafka | 内存队列 |
| --- | --- | --- | --- | --- |
| **吞吐量** | 高（十万级 QPS） | 中（万级 QPS） | **极高（百万级，靠分区水平扩展）** | 极高（受限于 CPU/内存） |
| **延迟** | **极低（亚毫秒级）** | **低（微秒/毫秒级，实时性强）** | 中（毫秒级，受批处理影响） | 极低（纳秒/微秒级） |
| **持久化** | 支持（RDB/AOF 异步） | 支持（磁盘） | **强支持（原生磁盘顺序写）** | 无 |
| **消息堆积** | 一般（受内存限制） | 中（堆积多时性能下降明显） | **极强（TB 级磁盘存储，性能稳定）** | 差（易 OOM） |
| **消息回溯** | 支持（按 ID/时间） | **不支持（传统队列模式下）** | **强支持（按 Offset/时间）** | 不支持 |
| **可靠性** | 中（AOF 丢数据风险） | **高（Confirm/确认机制成熟）** | **极高（多副本 + 强一致性配置）** | 低 |
| **运维复杂度** | 低（运维 Redis 即可） | 中（Erlang 环境，集群管理） | 高（依赖 ZK 或 KRaft） | 极低 |
| **适用场景** | 轻量级、低延迟、已有 Redis | **复杂路由、高可靠性、金融业务** | **大数据、日志聚合、高吞吐流处理** | 进程内解耦、极致性能要求 |

### 总结

**回到最初的问题：Redis 到底能不能做 MQ？**

- **如果业务简单、量小、追求极致性能** ，且能容忍极小概率的数据丢失，使用 **Redis Stream** 是最优解，因为它省去了部署维护 MQ 的成本，可以复用现有的 Redis 组件（大部分需要用到 MQ 的项目，通常都会需要 Redis）。
- **如果是金融级业务、海量数据、需要严格保证不丢消息** ，必须选择 **Kafka、RabbitMQ** 等更成熟的 MQ。

更多 Redis 高频知识点和面试题总结，可以阅读笔者写的这几篇文章：

- [Redis 常见面试题总结（上）](https://javaguide.cn/database/redis/redis-questions-01.html "Redis 常见面试题总结（上）") （Redis 基础、应用、数据类型、持久化机制、线程模型等）
- [Redis 常见面试题总结(下)](https://javaguide.cn/database/redis/redis-questions-02.html "Redis 常见面试题总结(下)") （Redis 事务、性能优化、生产问题、集群、使用规范等）
- [如何基于Redis实现延时任务](https://javaguide.cn/database/redis/redis-delayed-task.html "如何基于Redis实现延时任务")
- [Redis 5 种基本数据类型详解](https://javaguide.cn/database/redis/redis-data-structures-01.html "Redis 5 种基本数据类型详解")
- [Redis 3 种特殊数据类型详解](https://javaguide.cn/database/redis/redis-data-structures-02.html "Redis 3 种特殊数据类型详解")
- [Redis为什么用跳表实现有序集合](https://javaguide.cn/database/redis/redis-skiplist.html "Redis为什么用跳表实现有序集合")
- [Redis 持久化机制详解](https://javaguide.cn/database/redis/redis-persistence.html "Redis 持久化机制详解")
- [Redis 内存碎片详解](https://javaguide.cn/database/redis/redis-memory-fragmentation.html "Redis 内存碎片详解")
- [Redis 常见阻塞原因总结](https://javaguide.cn/database/redis/redis-common-blocking-problems-summary.html "Redis 常见阻塞原因总结")

我的 [《SpringAI 智能面试平台+RAG 知识库》](https://javaguide.cn/zhuanlan/interview-guide.html) 项目就是用的 Redis Stream 作为消息队列。在我的项目的场景下，它几乎是最合适的选择，完全够用了。

![系统架构图](https://oss.javaguide.cn/xingqiu/pratical-project/interview-guide/interview-guide-architecture-diagram.png)

![AI 智能面试平台效果展示](https://oss.javaguide.cn/xingqiu/pratical-project/interview-guide/page-resume-history.png)