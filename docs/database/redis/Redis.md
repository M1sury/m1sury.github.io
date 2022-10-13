## Redis基础

### 什么是Redis？

[Redis](https://redis.io/) 是一个基于 C 语言开发的开源数据库，与传统数据库不同的是 Redis 的数据是存在内存中的，读写速度非常快，被广泛应用于缓存方向。并且，Redis 存储的是 KV 键值对数据

**为了满足不同的业务场景，Redis内置了多种数据类型实现**

并且 Redis没有外部依赖，官方推荐生产环境使用 Linux 部署Redis



### Redis为什么这么快？

* Redis 基于内存，内存的访问速度是磁盘的上千倍
* Redis 基于 Reactor 模式设计开发了一套高效的事件处理模型，主要是**单线程事件循环**和**IO 多路复用**
* Redis 内置了多种优化后的数据结构实现，性能高



### Redis除了做缓存，还可以做什么？

* **分布式锁：**通过 Redis 来做分布式锁是一种比较常见的方式。相关阅读：[分布式锁的解决方案-Redssion](https://mp.weixin.qq.com/s/CbnPRfvq4m1sqo2uKI6qQw)
* **限流：**一般是通过 Redis + Lua 脚本的方式来实现限流。相关阅读：[Redis分布式限流器](https://mp.weixin.qq.com/s/kyFAWH3mVNJvurQDt4vchA)

* **消息队列：**Redis 自带的 list 数据结构可以作为一个简单的队列使用。Redis5.0 中增加 Stream 类型的数据结构更加适合用来做消息队列。类似于 kafka，有主题和消费组的概念，支持消息持久化以及 ACK 机制
* ……





### Redis 数据结构

> 5种基础数据结构：String、List、Set、Hash、Zset
>
> 3种特殊数据结构：HyperLogLogs、Bitmap、Geospatial

#### String