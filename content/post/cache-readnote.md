---
# 常用定义
title: "缓存阅读笔记"           # 标题
date: 2019-04-26T10:01:23+08:00    # 创建时间
lastmod: 2019-04-26T10:01:23+08:00 # 最后修改时间
draft: false                       # 是否是草稿？
tags: ["cache"]  # 标签
categories: ["tech"]              # 分类
author: "zhouxwyeah"                  # 作者
description: "缓存设计笔记"
# 用户自定义
# 你可以选择 关闭(false) 或者打开(true) 以下选项
comment: true   # 关闭评论
toc: true       # 关闭文章目录
# 你同样可以自定义文章的版权规则
reward: false	 # 关闭打赏
mathjax: true    # 打开 mathjax

---

> 某位大佬说过，性能优化就是找到性能热点，优化算法，如果无法再优化，空间换时间，而空间换时间，一般就是缓存

本文关于缓存方面的阅读笔记，希望通过本文理清关于缓存设计的一些思路，并参考当前一些java cache的现有方案。

#### 缓存的要求

* 支持高并发,良好的并发性能
* 内存大小可限制
* 扩展性强，如果需求增加，可以方便的scale
* 命中率高
* 处理好雪崩，穿透等特殊情况
* 分布式缓存

#### 缓存算法

重点在于过期策略，尝试判断数据最大可能被再次访问，也就是最大程度的提升缓存命中率

* FIFO 先进先出  无法保证命中率

* LRU 算是最通用的做法，默认认为缓存命中率和使用历史关联，每次读写都影响排名。参考实现： 一个双向链表存储数据，一个HashMap存储数据，双向链表做好淘汰、升级维护,每次插入放在头部，每次淘汰删除尾部，通过HashMap获取数据，操作可以做到O（1）

* LFU 最小频率优先淘汰，HashMap 存储数据,维护F的成本比R高，由多个二维链表维护LFU数据,操作次数一个NodeQueue,Queue中按照数据操作时间排序，做到O(1)

* W-LRU(Weighted Least Recently Used ) 现代缓存算法，都在LRU和LFU上做结合。 W—LRU利用Count-Min Sketch记录访问频率（类似布隆过滤器）

* ARC [ARC缓存替换算法](https://zhuanlan.zhihu.com/p/60916764) ，使用两个LRU，L1存放最近访问一次的数据，L2存放2次以及以上的数据，由于完全LFU成本较高，所以使用2次近似的做法。L1+L2 =2C ，目的维持 L1 = L2 =C ，如果L1 < C ,则优先淘汰L2给L1腾位置

  以下各个算法的效率比对：


​		![命中率](https://lh5.googleusercontent.com/FMyO_6DWOaAU-ulD38My9G-KpmYdAe8IQupR0n9MUEDVlDJ9xjvnJtccy2lms-rzQ3bPkbB773lPsCS67WgvOrLn-LpVBkVNCjfavzi4QD3BSNwWp0gGQEI_21rfhKytOLp-N2A)

#### 一些做法

* ConcurrentHashMap   无法支持内存可控，没有刷新和过期策略，并发通过lock-strip解决，16把锁，简单用在一些大小可控的数据场景，比如常量、指定参数等，比较简单

* LRMMap 基于LinkedHashMap,LRU 

* Ehcache 老牌缓存，可选FIFO，LRU，LFU，支持持久化，但是命中率不高

* Guava Cache    也是分段锁，读写锁分离，支持过期时间，支持自动刷新。Guava的分段是动态计算的，每个端自己会计算淘汰过程，可能出出现随机淘汰。用户通过HASH查找到分段，然后判断元素释放过期，过期则重新load。这里过期没有通过扫描来做，而是通过读写操作完成过期处理，避免后台线程扫描的的全局锁。在Guava cache中，key和value都能进行虚引用的设定，在Segment中的两个队列用来记录被回收的引用，其中每个队列记录了每个被回收的Entry的hash，这样回收了之后通过这个队列中的hash值就能把以前的Entry进行删除。

* Caffeine [现代高级缓存](http://highscalability.com/blog/2016/1/25/design-of-a-modern-cache.html),号称满足所有条件，良好的缓冲命中率和并发性能。

  淘汰驱逐策略，使用W-TinyLFU算法，如果它的频率高于必须被逐出的条目，则允许新条目。入口窗口不是立即过滤，W-TinyLFU使用分段LRU（SLRU）策略进行长期保留。一个条目从试用段开始，在随后的访问中，它被提升到受保护的段（上限为80％容量）。当受保护的段满时，它会逐出进入试用段，这可能会触发暂停试用。这样可以确保保留具有较小重用间隔（最热）的条目，并且较少重复使用（最冷）的条目符合驱逐条件。

  三个LRU队列，Eden队列 最安逸的队列，新的记录，Probation 缓刑队列，数据相对比较了冷，马上就要被淘汰，Protected 保护队列，短期不会被删除

  [队列情况](https://user-gold-cdn.xitu.io/2018/8/16/1654222b063487e1?imageView2/0/w/1280/h/960/ignore-error/1)

  

  并发方面，避免使用锁或者分段锁，参考数据库理论，每次写操作都是一个commitlog,写操作避免同步操作，使用异步批量执行. 记录缓存操作日志到buffer,在必要的调度处理这些事件，这个策略同样会有锁，但是把锁竞争的转换到了日志buffer。为了解决这个问题，Caffeine使用了缓存分离的方式，将读buffer和写buffer分离，一个操作记录，首先通过HASH计算绑定对应的线程，记录到指定的ring buffer，随着竞争增加，ring buffer的数量也会增加。

  当一个环形缓冲区已满时，将调度异步排空，并放弃对该缓冲区的后续添加，直到空间可用为止。当由于完整缓冲区而未记录访问时，缓存的值仍然返回给调用者。政策信息的丢失没有产生有意义的影响，因为W-TinyLFU能够识别我们希望保留的热门条目。通过使用特定于线程的哈希而不是密钥的哈希，缓存通过更均匀地分散负载来避免流行条目引起争用。

  在写入的情况下，使用更传统的并发队列，并且每个更改都安排立即消耗。虽然数据丢失是不可接受的，但仍有一些方法可以优化写缓冲区。两种类型的缓冲区都由多个线程写入，但在给定时间仅由单个线程使用。这种多生产者/单一消费者行为允许采用更简单，更有效的算法。

  ![并发效果](https://lh5.googleusercontent.com/Zrb2daVCRylwYZpSKykg1HVxCwsJI6N4sB7KQZ6wTfVdhdZNf75PNi38kO9BKMGn0PW5hHgaEDg2JfFtqbAPU3r4kQZ1toHJ0k7b7gTFII0pXbAJw0pbJ1oPjmqL6ZVEKBHSzLY)

* Redis /Memorycache   分布式缓存，一般都是key-value方式，redis支持更多的数据结构。

#### 多级缓存

​		本地缓存—远程缓存----DB

*  查询本地缓存，如果存在返回，不存在，查询远程缓存，并更新本地缓存

* 远程缓存存在，则返回，更新本地缓存，不存在则查询DB，并更新远程缓存和本地缓存

  ![多级缓存](https://user-gold-cdn.xitu.io/2018/8/21/1655d35701c24b7f?imageView2/0/w/1280/h/960/ignore-error/1)

#### 缓存更新策略

* 先更新数据库，然后删除缓存

  如果删除失败，可以异步继续删除

#### 缓存问题

* 缓存穿透

超时NULL缓存、布隆过滤器（位图）

* 缓存击穿

热点缓存瞬间失效，加锁，由一个线程进行更新

或者不失效，异步更新

* 缓存雪崩

大量缓存同时失效，设置random失效，不要在同一时间失效

#### JSR-107  JCache

现在大部分的缓存框架，都会参考JSR-107的JCache接口，我们熟悉一下JCache操作规范，方便后续我们使用各类缓存。

JCache API 解决以下问题：

* 提供应用缓存功能，特别是java对象缓存
* 定义通用的缓存概念和工具
* 最少相关的概念数量，减少程序员的负担
* 最大化的考虑缓存使用和实现
* 支持进程间和分布式缓存
* 支持value和引用缓存
* 定义JSR-175，缓存相关的注解

JCache规范不关注：

* 资源和内存约束
* 存储和拓扑
* 管理和监控
* 安全
* 资源同步

JCache API 顶层包名为 javax.cache，提供5个通用的接口

* CachingProvider 提供机制配置管理多个CacheManager，一个应用运行时由0-N个CachingProvider
* CacheManager   提供机制配置管理多个Cache，一个CacheManager属于一个CachingProvider
* Cache Map-Like数据结构临时缓存k-v数据，Cache属于一个CacheManager
* Entry Cache存储的V
* ExpiryPolicy  每个Entry都会定一个生存周期

还有一些其他的概念

* ClassLoader 支持Cache指定类
* Configuration 配置缓存策略类型、过期策略、统计
* CacheProcess 缓存处理器
* CacheEntryListener
* CacheEntryFilter
* CacheWriter
* CacheLoader

通用注解

* @CacheDefault 
* @CacheResult  表示方法返回值会被缓存，KEY通过方法参数自动生成
* @Cacheput  表示方法返回值会被缓存，CACHE-KEY需要自己指定
* @CacheRemove 方法会移除缓存
* @CacheRemoveALL  清理所有缓存
* 管理MutableConfiguration 