---
title: 浅谈Redis的结构和应用
date: 2020-02-16 09:21:08
tags: [Redis]
categories: Redis
---

#### 前言

Redis我想大家都很熟悉，Redis是Key-Value存储的，它是一个完全内存存储，当然Redis也可以进行持久化（RDB，AOF），持久化是在硬盘上的。Redis最常用是用Redis去做缓存，也可以做消息队列和数据库，内置Lua脚本引擎（分布式锁原子操作），当Redis内存使用完会使用默认LRU算法清除Key，同时还有Sentinel的高可用集群(HA)和Cluster集群实现分片存储和内存扩大（Hash一致性算法：扩容或者宕机后不会导存储发生变化即大量key由于Hash改变导致失效，Hash存储和之前保持一致）。

Redis常用的数据结构大家也都熟悉，Redis常用数据结构我想大家也都知道，除了最常用的String，List，Set，Hash，Zset；在Redis2.2版本引进BitMap，2.8版本引入HyperLogLog数据结构，3.2版本引入了存放地理信息的GEO数据结构和指令；5.0版本还引入Stream有点像抽象的日志数据结构，像日志文件一样，通常是以append only模式下打开的文件来实现的，不过这里我们久不涉及了。丰富了Redis的数据类型也增加Redis解决问题的其他能力，所以直到现在我所了解的Redis主要有8种数据类型，后面如果增加我们在增加。

Redis一般都是暂时性存储，不会做永久性存储，毕竟内存的还是有限的，同时内存的价格也比较贵，所以在使用Redis都会设置Key的过期时间，无论这个过期Key有多长（一般由业务决定）。这样不会导致Redis被一些无用的Key所撑爆，尽管Redis已经有LRU的淘汰策略。

我们主要讨论时redis4.0版本的数据结构类型。5.0版本先不涉及，现在好多redis版本还是在3.0版本。

#### Redis常用的5种数据结构

redis常用的五种类型就是String，List，Set，Hash，Zet。这里面Zet应该也不算是常用。这里最常用的应该就是String。

##### String

String是Redis最基本的数据结构，它是最常用的Redis数据结构，当你想要缓存一个值或者字符串的时候String最好的选择。关于String操作的指令也很简单，主要就是SET，GET这两种命令。

```
redis 127.0.0.1:6379> SET w3ckey redis 
OK 
redis 127.0.0.1:6379> GET w3ckey 
"redis"
```

Redis除了这两种常用的指令，还有一些可以其他功能的指令。

`SETNX key value`只有在 key 不存在时设置 key 的值。这个主要用于做分布式锁，在抢占资源的时候，一个Key设置好了就不允许其他相同的Key在设置进来，实现了互斥。不过老版本的一般用`SETNX`和`EXPIRE`这两个指令来实现这样就会有两个指令不同步问题，因为两条指令不是原子操作有有的时候就会与异常情况要处理。现在Redis的Redisson里面有自带分布式锁，它可以通过Lua脚本实现分布式锁。就是jedis也提供了SET一个函数，可以指定NX和过期时间的参数。

```java
//Redis通过Lua脚本设置锁
<T> Future<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId,
                RedisStrictCommand<T> command) {
    internalLockLeaseTime = unit.toMillis(leaseTime);
    // 4.使用 EVAL 命令执行 Lua 脚本获取锁
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
              "if (redis.call('exists', KEYS[1]) == 0) then " +
                  "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
              "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                  "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
              "return redis.call('pttl', KEYS[1]);",
                Collections.<Object>singletonList(getName()), internalLockLeaseTime,
                        getLockName(threadId));
```

1、如果通过 `exists` 命令发现当前 key 不存在，即锁没被占用，则执行 `hset` 写入 Hash 类型数据 **key:全局锁名称**（例如共享资源ID）, **field:锁实例名称**（Redisson客户端ID:线程ID）, **value:1**，并执行 `pexpire` 对该 key 设置失效时间，返回空值 `nil`，至此获取锁成功。

2、如果通过 `hexists` 命令发现 Redis 中已经存在当前 key 和 field 的 Hash 数据，说明当前线程之前已经获取到锁，因为这里的锁是**可重入**的，则执行 `hincrby` 对当前 key field 的值**加一**，并重新设置失效时间，返回空值，至此重入获取锁成功。

3、最后是锁已被占用的情况，即当前 key 已经存在，但是 Hash 中的 Field 与当前值不同，则执行 `pttl` 获取锁的剩余存活时间并返回，至此获取锁失败。

Jedis 中是这种SET key value [EX seconds] [PX milliseconds] [NX|XX]。

```java
 /**
     * SET key value [EX seconds] [PX milliseconds] [NX|XX]
     * 将字符串值 value 关联到 key 。
     * 如果 key 已经持有其他值， SET 就覆写旧值，无视类型。
     * 对于某个原本带有生存时间（TTL）的键来说， 当 SET 命令成功在这个键上执行时， 这个键原有的 TTL 将被清除。
     * 可选参数
     * 从 Redis 2.6.12 版本开始， SET 命令的行为可以通过一系列参数来修改：
     * EX second ：设置键的过期时间为 second 秒。 SET key value EX second 效果等同于 SETEX key second value 。
     * PX millisecond ：设置键的过期时间为 millisecond 毫秒。 SET key value PX millisecond 效果等同于 PSETEX key millisecond value 。
     * NX ：只在键不存在时，才对键进行设置操作。 SET key value NX 效果等同于 SETNX key value 。
     * XX ：只在键已经存在时，才对键进行设置操作。
     */
    @Test
    public void SET() {
        //设置不存在key为name时设置并在15秒后过期,一般这就是redis锁操作实现。
        jedis.set(KEY, VALUE, "NX", "EX", 15);
        out(jedis.get(KEY));
    }

```

我一般常用的是jedis的SET方法，这样比较简单。不过上面两种都需要Redis2.6版本以上，不过现在Redis已经5.0版本都有了。如果还是2.6版本那就赶紧升级一下，还可以体验一下其他的新功能。

`INCR key `将 key 中储存的数字值增一。

```
redis> SET page_view 20
OK
redis> INCR page_view
(integer) 21
redis> GET page_view    # 数字值在 Redis 中以字符串的形式保存(底层数据结构是sds)
"21"
```

  ` INCRBY key increment`将 key 所储存的值加上给定的增量值（increment） 。指定增加的步长

```
edis> SET rank 50
OK
redis> INCRBY rank 20
(integer) 70
redis> GET rank
"70"
 
# key 不存在时
redis> EXISTS counter
(integer) 0
redis> INCRBY counter 30
(integer) 30
redis> GET counter
"30"

# key 不是数字值时
redis> SET book "long long ago..."
OK
redis> INCRBY book 200
(error) ERR value is not an integer or out of range
```

这两个指令在生成唯一编号的时候会很好用，也就是一个发号器，可以生成订单编号（这样保证的唯一性和顺序性对于存储时有很大好处），当然可以用数据库和雪花算法等等。这种增长值限制在 64 位(bit)有符号数字表示之内。

同样也有`DECR Key` 和`DECRBY key decrement`这种也可以用在分布式的并发控制器。做一个分布式的`CountDownLatch` 做资源的控制，当Key 减小到0时候才会一起执行。

使用`INCRBY`可以做原子计数器，对于某个用户请求做记录，这样简单知道这个用户请求多少次网站。同样根据这个也可以去做限流控制，对于某个用户的频繁操作做熔断。

同样String还有很多其他的指令操作，不过我这边就不一一列举，可以到Redis中文网去看String的指令操作(https://www.redis.net.cn/tutorial/3508.html)。

String的value是二进制安全的，你可以存二进制。在Redis中字符串类型的Value最多可以容纳的数据长度是512M。肯定有人会问Redis中可以存多少个Key，目前Redis可以存2^32个Key。

##### List

Redis列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素导列表的头部（左边）或者尾部（右边）.一个列表最多可以包含 2^32 - 1 个元素 (4294967295, 每个列表超过40亿个元素)。

List常用的指令有`LPUSH`、`LPOP`、`LLEN`、`LPUSHX`、`LRANGE`、`LSET`、`LINDEX`、`BLPOP`这些常用的指令。我这边展示很多都是Left也就是从List左侧进行操作，在Redis中List更像一个队列，而且是一个双向的队列，它还有Right相关的操作，例如`RPOP`、`RPUSH`等等都是操作列表尾部的数据。

```
redis 127.0.0.1:6379> LPUSH w3ckey redis //将redis从头部插入数据向w3ckey列表。
(integer) 1
redis 127.0.0.1:6379> LPUSH w3ckey mongodb //将mongodb从头部插入数据向w3ckey列表。
(integer) 2
redis 127.0.0.1:6379> LPUSH w3ckey mysql //将mysql从头部插入数据向w3ckey列表
(integer) 3
redis 127.0.0.1:6379> LRANGE w3ckey 0 10 //这个地方也可以0 -1 显示全部，展示0-10的数据 
1) "mysql"
2) "mongodb"
3) "redis"
redis 127.0.0.1:6379> RPOP w3ckey //从尾部弹出一个元素
"redis"
redis 127.0.0.1:6379> LRANGE w3ckey 0 -1 //展示全部数据
1) "mysql"
2) "mongodb"
redis 127.0.0.1:6379> RPUSH w3ckey hbase //从尾部插入hbase数据
(integer) 3
redis 127.0.0.1:6379> LRANGE w3ckey 0 -1 
1) "mysql"
2) "mongodb"
3) "hbase"
redis 127.0.0.1:6379> LPOP w3ckey //从头部弹出数据
"mysql"
redis 127.0.0.1:6379> LRANGE w3ckey 0 -1
1) "mongodb"
2) "hbase"
redis 127.0.0.1:6379> LLEN w3ckey //显示列表长度
(integer) 2
redis 127.0.0.1:6379> LSET w3ckey 1 kafka //像位置为1的地方插入数据kafka
OK
redis 127.0.0.1:6379> LRANGE w3ckey 0 -1 //从结果我们可以看出来hbase被覆盖。0是mongodb
1) "mongodb"
2) "kafka"
redis 127.0.0.1:6379> LINDEX w3ckey 1 //获取索引下标是1的数据
"kafka"
redis 127.0.0.1:6379> LRANGE w3ckey 0 -1 //从结果我们可以看出来这个数据还在集合里面，没有被弹出
1) "mongodb"
2) "kafka"
```

List可以用于分布式队列，或者是栈。这样就可以多个应用共同消费不会有重复消费的情况。如果是做一个队列使用的是`LPUSH`+`RPOP`这两个指令从头（左端）插入，从尾部（右端）弹出。如果做一个栈的化，使用`LPUSH`+`LPOP`这种一般会使用即先进后出，从头部（左侧）插入，头部弹出。

这种可以使用做微博的TimeLine或者是报警，存储的消息队列（根据生产者和消费者模型）。

##### Hash

Redis hash 是一个string类型的field和value的映射表，hash特别适合用于存储对象。Redis 中每个 hash 可以存储 2^32 - 1 键值对（4294967295）

Hash我像大家应该都不陌生，Hash在计算机里面是经常见到的一个词汇，所以Hash这个比较容易理解的。Redis里的Hash和我们平时看到的HashMap差不多。

常用的指令有`HSET Key Field Value`、`HMSET Key [Filed1 Value1] Field2 Value2`、`HGET Key Field`、`HGETALL Key` 、`HLEN Key`、`HMGET Key [Field1] Field2`、`HDEL Key [Field1] Field2`、`HVALS Key` 、`HSETNX Key Field Value`、`HEXISTS Key Field`等常用指令。

```
redis 127.0.0.1:6379> HMSET w3ckey name "redis tutorial" description "redis basic commands for caching" likes 20 visitors 23000   
OK   //设置Hash 键 name 值 redis tutorial 键 likes 值 20 等等
redis 127.0.0.1:6379> HGETALL w3ckey 获取所有key和value
 1) "name"
2) "redis tutorial"
3) "description"
4) "redis basic commands for caching"
5) "likes"
6) "20"
7) "visitors"
8) "23000"
redis 127.0.0.1:6379> HGET w3ckey name //获取Hash种单个key里面的值
"redis tutorial"
redis 127.0.0.1:6379> HSET w3ckey address home //设置Hash中 address的值是home
(integer) 1
redis 127.0.0.1:6379> HLEN w3ckey //获取Hash的大小
5
redis 127.0.0.1:6379> HVALS w3ckey //获取Hash中所有的value值
1) "redis tutorial"
2) "redis basic commands for caching"
3) "20"
4) "23000"
5) "home"
redis 127.0.0.1:6379> HMGET w3ckey name address //获取Hash中 key是name key是address的值
1) "redis tutorial"
2) "home"
redis 127.0.0.1:6379> HDEL w3ckey description //删除Hash中的description
(integer) 1
redis 127.0.0.1:6379> HGETALL w3ckey //description被删除了
1) "name"
2) "redis tutorial"
3) "likes"
4) "20"
5) "visitors"
6) "23000"
7) "address"
8) "home"
redis 127.0.0.1:6379> HEXISTS w3ckey name //Hash中存在name这个key，存在返回1不存在返回0
(integer) 1
redis 127.0.0.1:6379> HSETNX w3ckey name redis-cluster //与SETNX类似，设置Key中不存在的值
(integer) 0 //失败返回0，已经存在name这个key
redis 127.0.0.1:6379> HSETNX w3ckey age 100 //设置成功age这个key
(integer) 1
redis 127.0.0.1:6379> HGETALL w3ckey
1) "name"
2) "redis tutorial"
3) "likes"
4) "20"
5) "visitors"
6) "23000"
7) "address"
8) "home"
9) "age"
10) "100"
redis 127.0.0.1:6379> HSCAN w3ckey 0 match a* count 100 //HSCAN 当key比较多可以使用迭代器
1) "0"                                                //方式去查找相关的key这个是匹配所有a
2) 1) "address"										  //打头的key。用于模糊查找key。
2) "home"			
3) "age"
4) "100"
```

根据Key获取值，主要用于缓存对象类的数据。一把都会把数据序列化，提高存储效率。举个例子可以存储购物车信息。以用户id为key，商品id为field，商品数量为value。

Hash的优点：同类数据归类整合储存，方便数据管理；相比String操作消耗内存与cpu更小；相比String储存更节省空间。

Hash的缺点：过期功能不能使用在field上，只能用在key上，Redis集群架构下不适合大规模使用。]

##### Set

Redis的Set是string类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据。Redis 中 集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。集合中最大的成员数为 2^32 - 1 (4294967295, 每个集合可存储40多亿个成员)。

常用的指令有`SADD`、 `SMEMBERS`、`SPOP`、`SCARD`这种和List类似的操做，还提供集合操作比如`SDIFF`、`SINTER`、`SUNION`这种差集，交集和并集的操作。这样可以在做大数据量的集合运算使用Redis。

```
redis 127.0.0.1:6379> SADD w3ckey redis
(integer) 1
redis 127.0.0.1:6379> SADD w3ckey mongodb
(integer) 1
redis 127.0.0.1:6379> SADD w3ckey mysql
(integer) 1
redis 127.0.0.1:6379> SADD w3ckey mysql
(integer) 0
redis 127.0.0.1:6379> SMEMBERS w3ckey
1) "mysql"
2) "mongodb"
3) "redis"
redis 127.0.0.1:6379> SCARD w3ckey
3
redis 127.0.0.1:6379>SPOP w3ckey
"mysql"
redis 127.0.0.1:6379> SMEMBERS wc
1) "333"
2) "211"
3) "redis"
4) "212"
5) "222"
6) "111"
redis 127.0.0.1:6379> SSCAN wc 0 match 2* count 10 //迭代器，用于模糊查找和匹配
1) "0"
2) 1) "222"
2) "211 
3) "212"

```

Set是和List差不多的数据结构，它比较像Java中的HashSet，和Java底层HashSet使用的是HashMap一样，这个Set使用的也是Hash。它和List比的好处就是它可以去重。比如你简单的统计用户访问量这种久可以用到Set这种数据结构。每一个用户来的时候我久在这个Set上加一。但是当数据量特别大的时候也不适合用Set去做，这种只是个临时方案。我们可以使用Redis不常用的数据结构比如一个BitMap或者HyperLogLog这种。

因为Set有很强的集合运算能力在很多社交应用会用到。比如使用使用sinter就可以计算出两个用户兴趣的交集。将两个用户和标签分别存储成两个Set（key是用户信息，set是标签数据）然后两个set做交集算出兴趣集合，当集合数量大于某个数就可以推荐好友了。不过用HaSet也可以做。

spop可以从集合中随机弹出元素，srandmember可以随机取出一个元素，但是不删除这个元素，这两个命令都可以实现不用需求的抽奖系统；还有微信，微博点赞功能。

微信抽奖小程序：1）点击参与抽奖加入集合 SADD key {userID}；2）查看参与抽奖所有用户 SMEMBERS key；3）抽奖count名中奖者 SRANDMEMBER key [count] / SPOP key [count]

微信微博点赞、收藏、标签：1）点赞；ADD like:{消息ID} {用户ID}；2）取消点赞 SREM like:{消息ID} {用户ID}；3）检查用户是否点过赞 SISMEMBER like:{消息ID} {用户ID}；4）获取点赞的用户列表 SMEMBERS like:{消息ID}；5）获取点赞用户数；SCARD like:{消息ID}

##### Zset（sorted set）

Redis的有序的集合。Redis 有序集合和集合一样也是string类型元素的集合,且不允许重复的成员。不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。有序集合的成员是唯一的,但分数(score)却可以重复。

在Zset中有三个区间要掌握，索引区间，字典区间，分数区间。分数区间时比较好算的，我们在设置值的时候都会指定分数，在一个就是索引区间，这个你返回值时候的下标，那个时索引。还有就是字典区间，这个时根据你设置的值中字典值。我们知道Zset中存储时String所以久可以根据字符串计算区间。（1，123，2）这种类型的区间。

一般指令时根据索引区间，LEX相关是字典区间，带SCORE的是分数区间。   

Zset里面有很多实用的指令`ZADD key [score value] score1 value1`、`ZRANGE key start end [WITHSCORES] `、`ZCARD key`、`ZINCRBY key increment member`、`ZRANGEBYLEX key min max`、`ZRANGEBYSCORE key min max`、`ZSCORE key member`、`ZREM key member [member ...]`、`ZREMRANGEBYSCORE key min max`、`ZREMRANGEBYRANK key start stop`、`ZREVRANGE key start stop`、`ZREVRANGEBYSCORE key max min`、`ZREVRANK key`、`ZSCAN key cursor [MATCH pattern] [COUNT count]`

后面还有`ZPOPMAX` 、`ZPOPMIN`、`BZPOPMAX`、`BZPOPMIN`弹出最大值，最小值和阻塞弹出最大值最小值。不过这个是在Redis5.0.0后面版本才可以使用。

```
redis 127.0.0.1:6379>ZADD w3ckeyz 1 redis //向集合中增加值和分数
(integer) 1
redis 127.0.0.1:6379> ZADD w3ckeyz 2 mongo
(integer) 1
redis 127.0.0.1:6379> ZADD w3ckeyz 3 mysql
(integer) 1
redis 127.0.0.1:6379> ZADD w3ckeyz 4 mysql
(integer) 0
redis 127.0.0.1:6379> ZADD w3ckeyz 2 hbase
(integer) 1
redis 127.0.0.1:6379> ZADD w3ckeyz 6 row
(integer) 1
127.0.0.1:6379> ZRANGE w3ckeyz 0 -1 //展示集合中所有的值
1) "redis"
2) "hbase"
3) "mongo"
4) "mysql"
5) "row"
127.0.0.1:6379> ZRANGE w3ckeyz 0 10 WITHSCORES //根据Scores来展示集合中所有的值，有序
1) "redis"
2) 1.0
3) "hbase"
4) 2.0
5) "mongo"
6) 2.0
7) "mysql"
8) 4.0
9) "row"
10) 6.0
127.0.0.1:6379> ZCARD w3ckeyz //获取集合中元素多少
5
127.0.0.1:6379>ZCOUNT w3ckeyz 0 2 //根据集合分数区间获取元素个数
3
127.0.0.1:6379> ZINCRBY w3ckeyz 4 mysql //mysql元素增加3
8.0
127.0.0.1:6379> ZRANGE w3ckeyz 0 10 WITHSCORES //mysql 分数已经变了
1) "redis"
2) 1.0
3) "hbase"
4) 2.0
5) "mongo"
6) 2.0
7) "row"
8) 6.0
9) "mysql"
10) 8.0
127.0.0.1:6379>ZLEXCOUNT w3ckeyz - + //- +是获取集合的长度。根据字符串长度，这个指令一般在分数相同
(integer) 5//时元素使用会比较好用。它里面都是按照Sting去存的1，128，2类似这样顺序。如果分数不同则不是很推荐使用。我没太看懂分数不同的返回值，这个可以使用ZRANGEBYLEX，这个指令来看。
redis 127.0.0.1:6379> ZADD myzset 0 a 0 b 0 c 0 d 0 e 0 f 0 g
(integer) 7
redis 127.0.0.1:6379> ZRANGEBYLEX myzset - [c //获取C之前的元素，包含c
1) "a"
2) "b"
3) "c"
redis 127.0.0.1:6379> ZRANGEBYLEX myzset - (c //获取C之前的元素，不包含c
1) "a"
2) "b"
redis 127.0.0.1:6379> ZRANGEBYLEX myzset [aaa (g //获取a到g之间的元素不包括g 
1) "b"
2) "c"
3) "d"
4) "e"
5) "f" 
redis 127.0.0.1:6379> ZLEXCOUNT myzset [b [f //获取b到f的元素，这里分数都一样比较好用
(integer) 5
redis 127.0.0.1:6379> ZRANGEBYSCORE w3ckeyz 0 2 //根据分数区间获取值
1) "mac"
2) "redis"
3) "hbase"
4) "mongo"
5) "windows"
redis 127.0.0.1:6379> ZRANGEBYSCORE w3ckeyz 0 2 WITHSCORES //根据分数区间获取值和分数
1) "mac"
2) 1.0
3) "redis"
4) 1.0
5) "hbase"
6) 2.0
7) "mongo"
8) 2.0
9) "windows"
10) 2.0
redis 127.0.0.1:6379> ZSCORE w3ckeyz mac //获取集合中值的SCORE
1.0
redis 127.0.0.1:6379> ZREM w3ckeyz windows //删除windows这个元素
1
redis 127.0.0.1:6379> ZREMRANGEBYSCORE w3ckeyz 6 8 //删除SCORE6-8这个区间的元素
2
redis 127.0.0.1:6379> ZRANGE w3ckeyz 0 -1 WITHSCORES //展示所有元素
1) "mac"
2) 1.0
3) "redis"
4) 1.0
5) "hbase"
6) 2.0
7) "mongo"
8) 2.0
redis 127.0.0.1:6379> ZREMRANGEBYRANK w3ckeyz 0 1 //删除索引0，1区间数据
(integer) 2
redis 127.0.0.1:6379> ZRANGE w3ckeyz 0 -1 WITHSCORES 
1) "hbase"
2) 2.0
3) "mongo"
4) 2.0
redis 127.0.0.1:6379> ZREVRANGE w3ckeyz 0 -1 WITHSCORES
1) "dog" 									//有序集中指定区间内的成员，通过索引，分数从高到底
2) 7.0
3) "cat"
4) 6.0
5) "hat"
6) 5.0
7) "linux"
8) 3.0
9) "mongo"
10) 2.0
11) "hbase"
12) 2.0
13) "mysql"
14) 1.0
15) "redis"
16) 0.0
17) "client"
18) 0.0
redis 127.0.0.1:6379> ZREVRANGEBYSCORE w3ckeyz 7 0
1) "dog"								//有序集中指定区间内的成员，通过SCORE，分数从高到底	
2) "cat"
3) "hat"
4) "linux"
5) "mongo"
6) "hbase"
7) "mysql"
8) "redis"
9) "client"
```

Zset可以使用做排行榜，这个是很容易的，因为每个元素都有score分数。Zset同样也可以用于社交和抽奖。而且有权重可以做设计推送权重比如多层次的好友，比如一级，二级，三级好友，这种就是通过Score去排序。使用Zset去做时间线（TimeLine） ，key是做的事情，Score是时间。用户投票，实时排行信息包含直播间在线用户列表，弹幕消息（可以理解为按消息维度的消息排行榜）。

#### Redis不常用的3种数据结构

##### HyperLogLog 

Redis HyperLogLog 是用来做基数统计的算法，HyperLogLog 的优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定 的、并且是很小的。

在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基 数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。

但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以 HyperLogLog 不能像集合那样，返回输入的各个元素。该结构使用一种近似值算法，标准误差0.81%。

HyperLogLog 在Redis实现也是有意思的，有时间可以写一个关于这种数据结构的解析，它其实和布隆过滤器有点像，都是通过位存储来实现大量数据的存储。（布隆过滤器是经过多次hash形成位存储），不过这两种方法导致计算方法会有一定偏差。千万级数据集下，随机数误差基本低于 1.5%， 完全不重复情况下误差也不超过 2%

常用的几种指令`PFADD` 、`PFCOUNT`、`PFMERGE`这几种指令

```
redis 127.0.0.1:6379> PFADD w3ckey redis //向key中增加值 
(integer) 1
redis 127.0.0.1:6379> PFADD w3ckey mysql4
(integer) 1
redis 127.0.0.1:6379> PFADD w3ckey mysql
(integer) 1
redis 127.0.0.1:6379> PFADD w3ckey hbase
(integer) 1
redis 127.0.0.1:6379> PFADD w3ckey mongo
(integer) 1
redis 127.0.0.1:6379> PFCOUNT w3ckey
(integer) 5
redis 127.0.0.1:6379> PFADD w3ckey mongo //向HyperLogLog中增加相同的值，返回结果为0
(integer) 0
redis 127.0.0.1:6379> PFCOUNT w3ckey //结果没增加值，说明HyperLogLog是一个无重复的集合。
(integer) 5
redis 127.0.0.1:6379> PFADD w3ckey1 db
(integer) 1
redis 127.0.0.1:6379> PFADD w3ckey1 redis
(integer) 1
redis 127.0.0.1:6379> PFADD w3ckey1 cache
(integer) 1
redis 127.0.0.1:6379> PFCOUNT w3ckey1
(integer) 3
redis 127.0.0.1:6379> PFMERGE w3ckey w3ckey1 //合并两个key中数据，两个key是有相同数据。
OK
redis 127.0.0.1:6379> PFCOUNT w3ckey //从结果我们能看出去重了，不是5+3 而是5+2
(integer) 7
redis 127.0.0.1:6379> PFCOUNT w3ckey1 //从结果看是合并到w3ckey上，原来的key没有变
(integer) 3

```

上面提到一个词是基数基数，简单介绍基数计数。

**基数计数(cardinality counting)**：通常用来统计一个集合中不重复的元素个数，例如统计某个网站的UV，或者用户搜索网站的关键词数量。数据分析、网络监控及数据库优化等领域都会涉及到基数计数的需求。 要实现基数计数，最简单的做法是记录集合中所有不重复的元素集合S，当新来一个元素x，若S中不包含元素x，则将x加入S，否则不加入，计数值就是S的元素数量。这种做法存在两个问题：

1. 当统计的数据量变大时，相应的存储内存也会线性增长
2. 当集合S变大，判断其是否包含新加入元素i的成本变大

HyperLogLog用于解决这种基数统计问题就很ok，它的误差率我们上面也提到在很大数据量时候，误差也是可以接受对于统计UV这种数据。还有就是IP访问量，计算搜索词的热度（每个搜索词的搜索个数，不同的用户）

##### BitMap

BitMap是位图，它是一串连续的二进制数字（0或1），每一位所在的位置为偏移(offset)，在BitM ap上可执行AND,OR,XOR以及其它位操作。不过Redis这里只存储位置信息和，位置的bit信息。也就是redis只知道哪个位置是0那个位置是1，这样比较白话。它存储位数2^32-1个。

基于最小的单位bit进行存储，所以非常省空间。设置时候时间复杂度O(1)、读取时候时间复杂度O(n)，操作是非常快的。二进制数据的存储，进行相关计算的时候非常快。方便扩容

Redis中BitMap被限制在512MB之内，所以最大是2^32位。建议每个key的位数都控制下，因为读取时候时间复杂度O(n)，越大的串读的时间花销越多。因为BitMap是存在Redis的String中

常用的指令就是指定和获取key某个位置的bit值。`SETBIT key  offset 1 or 0`、`GETBIT key offset` 、`BITCOUNT key`、`BITOP`

```
redis 127.0.0.1:6379> SETBIT hello 100 1 //设置hello这个key 第100位为1
0
redis 127.0.0.1:6379> SETBIT hello 200 1 //设置hello这个key 第200位为1
0
redis 127.0.0.1:6379> GETBIT hello 100  //获取hello这个key 第100位置信息，结果返回是1
1
redis 127.0.0.1:6379> GETBIT hello 200  //获取hello这个key 第200位置信息，结果返回是1
1
redis 127.0.0.1:6379> GETBIT hello 10 //获取hello这个key 第10位设置信息，结果返回是0，没有初始化
redis 127.0.0.1:6379> BITCOUNT hello //计算hello这个key 有多少个位置被设置值
2
redis 127.0.0.1:6379> SETBIT bits-1 0 1        //bits-1 = 1001
0
redis 127.0.0.1:6379> SETBIT bits-1 3 1
0
redis 127.0.0.1:6379> SETBIT bits-2 0 1        // bits-2 = 1011
0
redis 127.0.0.1:6379> SETBIT bits-2 1 1
0
redis 127.0.0.1:6379> SETBIT bits-2 3 1
0
redis 127.0.0.1:6379> BITOP AND and-result bits-1 bits-2
1
redis 127.0.0.1:6379> GETBIT and-result 0     // and-result = 1001
1
redis 127.0.0.1:6379> GETBIT and-result 1
0
redis 127.0.0.1:6379> GETBIT and-result 2
0
redis 127.0.0.1:6379> GETBIT and-result 3
1
```

一般BitMap用作大数据处理，比如之前说的日活统计，重复点赞等。不过BitMap存储的是准确的数据。（不像HyperLogLog是由概率问题）。

可以根据用户Id然后hash一个位置，在这个位置上设置位1，表明一个用户来过。可以统计UV。同样也可以去做他的相关额基数统计，不过这种会随着量增大，会增大，但也不会很大和HyperLogLog比消耗还是比较大的，但是与Set像比还是可以的，比较省内存的，毕竟内存也是钱啊，而且它的数据是准的不会有误差。

##### Geo

Geo相关的操作指令和存储，是Redis专门为地理位置信息服务的，它主要把地理位置信息按照经纬度存储起来，然后去计算地理位置相关的操作，比如计算两个点的距离，根据具体坐标获取范围内的经纬度坐标等；它主要用于LBS相关的系统。使用Geo相关的指令和操作要在Redis版本3.2以上。

`常用的Geo指令 geoadd`、`geopos`、`geodist`、`georadius`、`georadiusbymember`、`geohash`等。根据这些指令我们就可以堆地理信息进行操作。然后根绝业务获取我们想要的数据。

```
redis 127.0.0.1:6379> GEOADD Sicily 13.361389 38.115556 "Palermo" 15.087269 37.502669 "Catania" 
(integer) 2 //设置Sicily key 两个地址Palermo和Catania
redis 127.0.0.1:6379> GEODIST Sicily Palermo Catania //计算Sicily key的两个节点的距离
"166274.15156960039"
redis 127.0.0.1:6379> GEORADIUS Sicily 15 37 100 km //获取经度15纬度37 100Km距离内的位置信息
1) "Catania"
redis 127.0.0.1:6379> GEORADIUS Sicily 15 37 200 km //获取经度15纬度37 200Km距离内的位置信息
1) "Palermo"
2) "Catania"
redis 127.0.0.1:6379> GEORADIUS Sicily 15 37 200 km WITHDIST //带出距离信息
1) 1) "Palermo"
   2) "190.4424"
2) 1) "Catania"
   2) "56.4413"
redis 127.0.0.1:6379> GEORADIUS Sicily 15 37 200 km WITHCOORD //带出距离信息和经纬度信息
1) 1) "Palermo"
   2) 1) "13.361389338970184"
      2) "38.115556395496299"
2) 1) "Catania"
   2) 1) "15.087267458438873"
      2) "37.50266842333162"
redis 127.0.0.1:6379> GEOPOS Sicily Palermo Catania NonExisting //获取节点的经纬度信息
1) 1) "13.361389338970184"
   2) "38.115556395496299"
2) 1) "15.087267458438873"
   2) "37.50266842333162"
3) (nil)
redis 127.0.0.1:6379> GEOADD Sicily 13.583333 37.316667 "Agrigento" //添加节点信息
redis 127.0.0.1:6379> GEORADIUSBYMEMBER Sicily Agrigento 100 km //返回这个位置信息100km内的位置信息
1) "Agrigento"
2) "Palermo"
```

用到的单位：m 表示单位为米。km 表示单位为千米。mi 表示单位为英里。ft 表示单位为英尺

我们除了使用附近的人，还有关于打车定位距离这些信息可以使用，还有很多应用场景。在内存中计算很快，特别是对那些实时性要求高的业务系统来说这些都是用的。

#### Redis是如何保证高性能

1、Redis是单线程，没有上下文切换，所以执行效率比较快。

2、Redis是纯内存数据库，除了持久化都在内存上操作，CPU之恶映射地址，直接读取数据，所以很快。

3、使用多路复用技术，可以处理并发的连接。非阻塞IO 内部实现采用epoll，采用了epoll+自己实现的简单的事件框架。epoll中的读、写、关闭、连接都转化成了事件，然后利用epoll的多路复用特性，绝不在io上浪费一点时间。

4、数据结构简单，对数据操作也简单，Redis中的数据结构是专门进行设计的。比如跳表，压缩表，还有SDS。

5、使用底层模型不同，它们之间底层实现方式以及与客户端之间通信的应用协议不一样，Redis直接自己构建了VM 机制 ，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求；

PS:Redis真的是单线程吗，有没有多线程的时候。Redis在进行进行过期Key删除的时候使用的是多线程。不过这不影响它插入和读取，我们最常用就是插入和读取。

#### Redis使用的一些小建议

1、所有Redis的key都要设置过期时间，这样就不会导致一些过期和垃圾的Key产生，大多数Redis都是做缓存，所以要保证时效性。

2、Redis的使用一定要分应用，应用隔离会使得Redis内存不会因为其他机器异常导致，本身服务不可用，因为内存毕竟不是一个可以压缩的资源。相对CPU来讲。

3、Redis的后面可以使用lettuce代替jedis，lettuce这个是使用netty做底层，而不是jedis使用的线程池去做。所以不会由线程释放问题。性能也会好很多

4、redis中两种持久化方式都要打开，RDB和AOF这样可以保证当Redis挂掉后可以根据RDB迅速把数据热起来，然后根据AOF在恢复近期的数据，进行修复。

5、Redis的集群由两种一种是Codis集中式和redis-cluster点对点模式两种，目前比较推荐使用redis-cluster做分片，首先这个是官方维护，后面可以一致使用。Codis的github已经不够活跃，同时codis好多指令也是不支持在分布式情况下。肯定有人会问哨兵模式会用的把，这两种无论那种在做HA（高可用）时候都会使用哨兵模式。即每个分片节点都是一个集群。

6、Redis的集群管理可以使用搜狐的缓存云cacheCloud进行管理，不过这个也不怎么活跃了。

7、当处理一些复杂业务和批量操作的时候使用lua或者pipleline。Redis后面瓶颈不是CPU而是网络，复杂业务频繁网络请求会降低Redis的使用性能。

8、Redis在进行大的Key存储一定要进行压缩或者序列化，不顾序列化选型也很关键，不要选效率比较低的。不然在高并发的时候你的所有CPU资源都用来序列化和反序列化这样导致你应用CPU过高的同时，频繁GC。

9、不要使用Key*这种扫描全部Key的指令，禁用效率低的指令。

#### 参考

[Redis](https://redis.io/)

[Redis中文网](https://www.redis.net.cn/)

[Try Redis](http://try.redis.io/)   这个上敲Redis命令真的爽，不用自己创建Redis

[RedisDOC](http://redisdoc.com/)

[神奇的HyperLogLog算法](http://www.rainybowe.com/blog/2017/07/13/%E7%A5%9E%E5%A5%87%E7%9A%84HyperLogLog%E7%AE%97%E6%B3%95/index.html)

[huangz/blog](https://blog.huangz.me/diary/2015/redis-geo.html)