---
title: 缓存与DB一致性问题
date: 2019-10-02 09:21:08
tags: [java,缓存]
categories: java
---

#### 缓存是什么

缓存（cache）：原始意义是指访问速度比一般RAM快的一种高速存储器，等等拿错剧本了。我们今天要说的是关于`JavaWeb`中使用的缓存，并不是计算机组成里面的缓存。在Java中常用`redis`数据库，`memcache`数据库等内存数据库去代替。因为它相对硬盘（持久化）数据库`MySQL`要快好多（当然`MySQL也`有自己的缓存，我们这边不展开）。所以一般我们使用这种数据库用于存放一些热点数据或者临时数据。

#### 缓存的作用

缓存的主要作用就是提高系统性能。特别是在热点数据比较多，而且读操作远大于写操作的时候。当一个读数据请求访问到应用，应用就不需要请求其他应用或者数据库。直接从缓存获取，从而提高了系统性能。但是从缓存中读取数据有好处，也有不好的，不好的就是可能会出现DB数据与缓存数据不一致的问题。所以如何保证缓存与DB数据致的问题，就是我们下面要聊的。

#### 缓存与DB读写的流程

缓存从数据库中读取数据，一般分三个步骤

1、先从缓存中读取，如果命中缓存，数据直接返回。

2、如果缓存中没有读取带数据，直接访问DB

3、从DB读取数据之后，回写到缓存中。

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/cache02.png)

在数据库写数据时操作过程

1、淘汰缓存中的数据记录

2、插入或者更新DB数据

3、在后续读取时出现了缓存命中失败没，然后回写缓存。

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/cache03.png)

#### 如何保证缓存与数据的一致性

##### 为什么在插入数据库之前要删除缓存呢？

如果先更新数据库（DB）成功了，但是缓存更新失败了，如果你没有其他缓存淘汰策略，那么读数据时会读取到缓存中的老数据，这样导致脏读的产生。

如果先删除缓存的数据，更新数据库（DB）的时候失败了。那么后面的读取从缓存获取数据会miss（不命中），会从数据库中重新获取数据，并插入到缓存中，保证了缓存与DB数据的一致。

同时也可以在，两个操作上加上事务，使更新DB失败的时候，进行回滚，保证了缓存不被清除。

##### 如果在多线程下，我们该如何保证缓存数据和DB数据的一致性

如果先删除缓存

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/cache05.png)

如果先更新数据库

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/cache06.png)

我们从图中能看出来，两种操作都会有问题。因为在多线程的情况下，无论对那个操作都有问题。问题不在顺序问题，而是不应该把两个操作（读操作和回写缓存的操作）放到一起去执行。(获取你想到，对缓存中每一个key都加上一把锁，这样是可以解决问题，但是这样是特别消耗性能，这样就不能够提高系统性能)

##### 读操作与回填缓存分离

解决这种问题，就是要把读数据库和回写缓存分开就可以了，因为回写缓存时在更新数据库之后。我们可以通过MQ将回写缓存和读取数据的操作分开。

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/cache07.png)

如果当application更新完数据库，但是还没有发送消息，宕机了。消息没发出去，导致缓存和数据库的数据不一致怎么办。

这个时候使用消息异步通知明显不够好，这个时候就可以mysql主从同步的时候使用的binlog同步来实现。application同步到到数据后把缓存更新掉。这样速度也快，同样如果application挂掉了，如果没同步后面也会同步的。肯定会有人说binlog同步数据成本很高，这样做不是要在应用里写好多代码实现起来肯定很困难。然而你肯定不是第一个碰到这个问题，所以肯定有人解决这个问题。

#### canal简介

alibaba开源了一个叫canal的框架，可以让你去伪装成一个数据库的从库去同步binlog。这样也就解决了问题

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/cache10.png)

主要用途是基于 MySQL 数据库增量日志解析，提供增量数据订阅和消费。基于日志增量订阅和消费的业务包括

- 数据库镜像
- 数据库实时备份
- 索引构建和实时维护(拆分异构索引、倒排索引等)
- 业务 cache 刷新（我们今天讲的）
- 带业务逻辑的增量数据处理

当前的 canal 支持源端 MySQL 版本包括 5.1.x , 5.5.x , 5.6.x , 5.7.x , 8.0.x

##### canal 工作原理

- canal 模拟 MySQL slave 的交互协议，伪装自己为 MySQL slave ，向 MySQL master 发送dump 协议
- MySQL master 收到 dump 请求，开始推送 binary log 给 slave (即 canal )
- canal 解析 binary log 对象(原始为 byte 流)

##### mysql主从同步原理

- MySQL master 将数据变更写入二进制日志( binary log, 其中记录叫做二进制日志事件binary log events，可以通过 show binlog events 进行查看)
- MySQL slave 将 master 的 binary log events 拷贝到它的中继日志(relay log)
- MySQL slave 重放 relay log 中事件，将数据变更反映它自己的数据

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/cache11.jpeg)

#### 总结

从上面的分析看，主要就是把查数据和回写缓存两个操作进行解耦合。这样不会让两个操作互相干扰。一开始通过mq的方式。现在可以使用同步binlog的方式防止，因为因消息发送失败导致一致性问题。同样使用这种框架会有问题，就是增加了系统的复杂性，也增加了排查问题的难度。