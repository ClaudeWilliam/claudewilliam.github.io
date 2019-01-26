---
title: 一致性Hash算法与Java的简单实现
date: 2018-10-05 09:21:08
tags: [java,算法]
categories: 编程
---

#### 重提Hash

你会发现，在编程的这个领域里面Hash是经常被提及的一个词，我们经常使用比如HashMap，HashSet等等都是与Hash有关，之前我也写过一篇关于HashMap的简介，就是简单的介绍一下Hash，那么我们现在又开始聊Hash一致性算法。这个也是和Hash有关。

之前我们提及Hash其实就是做一个映射，把一个比较大的集合，或者一个比较复杂的集合，映射到一个比较简单的集合。这个过程叫Hash，这样我们在这个简单的集合实现一些运算比较容易。这是一种比较简单的说法，但是实际上真正的Hash要比这个复杂，Hash算法叫散列值算法。

散列函数是任何函数可被用于映射数据任意大小的为固定大小的数据。散列函数返回的值称为散列值，散列码，摘要或简单的散列。在HashMap中一般这个散列码就是数组中数组下标。但是这个散列码也就是我们经常说的HashCode。估计这个时候有人会问hashCode一直的两个对象相等吗。一般来讲HashCode相等的两个对象一般不等，而相等的两个对象HashCode一定相等。毕竟Hash算法一般是有冲突的，所以HashMap中是使用数组+链表(拉链)的形式来解决冲突，具体在这里不介绍了。

一般我们都使用的是什么Hash算法，一般使用都是取模的方实现Hash映射，这种方式一般容易理解，而且简单暴力。而且好多数据结构也都这样的实现，比如HashMap。

简单hash算法在实现上为：目标 target = hash（key）/ mod（质数）。key就是一个关键字的Code，target就是下标位置。质数一般就是数组的大小。（这个时候有人肯定会问不是质数好不好，这里我们先留意疑问，我们在下面去解释这个为什么是质数，而且这个也要涉及一些枯燥的概念）

#### 如何判断一个Hash算法好坏（在分布式环境下）

##### 平衡性(Balance)

平衡性是指哈希的结果能够尽可能分布到所有的缓冲中去，这样可以使得所有的缓冲空间都得到利用。很多哈希算法都能够满足这一条件。

##### 单调性(Monotonicity)

单调性是指如果已经有一些内容通过哈希分派到了相应的缓冲中，又有新的缓冲加入到系统中。哈希的结果应能够保证原有已分配的内容可以被映射到原有的或者新的缓冲中去，而不会被映射到旧的缓冲集合中的其他缓冲区。

##### 分散性(Spread)

在分布式环境中，终端有可能看不到所有的缓冲，而是只能看到其中的一部分。当终端希望通过哈希过程将内容映射到缓冲上时，由于不同终端所见的缓冲范围有可能不同，从而导致哈希的结果不一致，最终的结果是**相同的内容被不同的终端映射到不同的缓冲区**中。这种情况显然是应该避免的，因为它导致相同内容被存储到不同缓冲中去，降低了**系统存储的效率**。分散性的定义就是上述情况发生的严重程度。好的哈希算法应能够尽量避免不一致的情况发生，也就是尽量降低分散性。（举个极端例子，如果你存100个对象的List，那么100个对象分部100台机器上，要对100台机器操作1次。这样效率极低，同时100台机器网络也会出现问题，但是如果50存放到A，50存放到B就会好很多，你就可以操作2台机器查找100条数据。其实就是和数组和链表差不多，在查找过程中由于数组是连续的内存随意可以快速的查找100个，而链表是100个分散的地址，只有找到当前，才能找到下一个。对于查找来说还是数组快一点也就是，相同（相关联）的数据都在一台机器上）

##### 负载(Load)

负载问题实际上是从另一个角度看待分散性问题。既然**不同的终端可能将相同的内容映射到不同的缓冲区**中，那么对于一个特定的缓冲区而言，也可能被不同的用户映射为不同的内容（导致一个节点对于多个终端访问，导致负载过高）。与分散性一样，这种情况也是应当避免的，因此好的哈希算法应能够尽量降低缓冲的负荷。（举一个极端的例子，就是一共有200个终端，存放200个数据，这个200数据中的部分数据都有存放到A机器上，那么A机器就相当有200负载，如果这有100个数据全部都存在B机器上，那么A，B的负载都是100，不是A是200，B也是200）

所以一个好的hash算法要有，高平衡（比较分散，不是集中），单调性（保证当有新的缓存加入时候不会大量失效，之前的数据），低分散（相同的数据尽量都在一起，提高系统存储效率），低负载（）

一般在分布式（缓存）系统中，通过这种方式来实现负载均衡（也就是集群中请求的分配）。一般这个mod就是机器的个数。这样可以快速的实现负载均衡的办法。但是这样有一个问题，就是分布式系统中有一台机器挂掉了，或者出现错误，或者说我临时增加或减少机器，那么这中间就会有大量的（缓存）数据失效，不得不重新的写入数据。（**简单Hash算法计算的值会依赖于mod。如果再加入一个分区则之前的hash映射都会失效，无法动态调整，严重违反单调性 **）因为你的mod变了，你求出来的余数也就会有很大的变化。为了解决这个问题，就出现了一致性Hash算法。它可以在最小程度上减小（缓存）数据的失效问题。同时通过虚拟节点的使用提高了平衡性，降低了分散性和负载。对于Hash算法来说

#### 什么是一致性Hash算法（Consistent hashing）

一致性哈希算法在1997年由麻省理工学院的Karger等人在解决分布式Cache中提出的，设计目标是为了解决因特网中的热点(Hot spot)问题，初衷和CARP（Common Access Redundancy Protocol 共用地址冗余协议）十分类似。一致性哈希修正了CARP使用的简单哈希算法带来的问题，使得DHT（分布式哈希）可以在P2P环境中真正得到应用。

一致哈希 是一种特殊的哈希算法。在使用一致哈希算法后，哈希表槽位数（大小）的改变平均只需要对  K/n 个关键字重新映射，其中 K是关键字的数量， n是槽位数量。然而在传统的哈希表中，添加或删除一个槽位的几乎需要对所有关键字进行重新映射。

一致哈希 是一种特殊的哈希算法。在使用一致哈希算法后，哈希表槽位数（大小）的改变平均只需要对  **K/n** 个关键字重新映射，其中 **K**是关键字的数量， **n**是槽位数量。然而在传统的哈希表中，添加或删除一个槽位的几乎需要对所有关键字进行重新映射。

一致哈希由MIT的Karger及其合作者提出，现在这一思想已经扩展到其它领域。在这篇1997年发表的学术论文中介绍了“一致哈希”如何应用于用户易变的分布式Web服务中。哈希表中的每一个代表分布式系统中一个节点，在系统添加或删除节点只需要移动 **K/n**项。

这是wiki上给的，我们看的比较懵逼，如果用图片在结合一个例子就很好的解释这个一致性Hash算法。

#### 一致性Hash算法，解决了什么问题

一致性哈希算法的基本原理，把数据通过hash函数映射到一个很大的环形空间里，Hash环上有0~2$^{32}$-1个槽（slot）。也就是一个圆，上面有2$^{32}$个点，这些点首尾相连，我们的所有的key就只能在这个圆上面映射。（一般来说这个数量也够了）

![](/img/hash/hash-01.png)

我们通过取余的计算方法，计算出key的位置，但是我们不把数据放在位置上，而是把数据存放到，最近的存储节点（也就是缓存服务器上）。也就是说你存放的数据的服务器也被映射到这个Hash环，上，然后计算你与存储节点的位置来定位你存储的位置。

A、B、C、D 4台缓存服务器通过hash函数映射环形空间上，数据的存储时，先得到一个hash值，对应到这个环中的每个位置，如缓存数据:K1对应到了图中所示的位置，然后沿顺时针找到一个机器节点A，将K1存储到A节点上，K2存储到A节点上，K3、K4存储到B节点上。

![](/img/hash/hash-02.jpg)

我们从图中可以看到，ABCD四个存储节点也映射到Hash环上了。然后他们每个人节点都可以控制自己的一块区域，这样可以实现负载均衡，假设这些点都均匀分布的话，相当这些槽就把数据给平分了。就只变动**1/n**的总数据，如果总数据是K个的话，当一个节点增加或者减少的时候会有也就是上面的**K/n**变动。

![](/img/hash/hash-03.png)

如果B节点挂机了，那么B节点的数据就会集中到C节点中。只会影响C节点，对其他的节点A，D的数据不会造成影响。这样是不是就可以了能，很明显这中间还是有很多问题。

首先就是数据是不是均匀，是不是数据都是均匀分在这个Hash环上，如果分布不均匀导致有的机器过载，但是有个机器却没有负载。在一点就是上面B节点宕机后，所有的数据都集中到B上会导致B集中了大量的集中数据。可能会导致“雪崩效应”

（B节点崩溃后，所有数据集中到A点，然后A点过载后也会崩掉。也就是由一个点的宕机导致整个集群垮掉的事情）。所以如何解决上面这两个问题。

答案就是增加节点个数，保证前面的数据均匀，但是一个节点可能挂掉也会导致雪崩效应，同时节点个数很多也是要钱的，所以我们就把这些节点变成虚拟节点。这些虚拟节点不存储数据，他会把数据给后面的真实节点。而且这些虚拟节点也可以与真实节点中间实现一个均匀（变动）的映射，不会使某个节点过载问题。

![](/img/hash/hash-05.png)

引入“虚拟节点”后，映射关系就从 {对象 ->节点 }转换到了 {对象 ->虚拟节点 }。图中的A1、A2、B1、B2、C1、C2、D1、D2都是虚拟节点，机器A负载存储A1、A2的数据，机器B负载存储B1、B2的数据，机器C负载存储C1、C2的数据。由于这些虚拟节点数量很多，均匀分布，提高了平衡性，因此不会造成“雪崩”现象。

所以一致性Hash算法提高了分布式系统的**性能**（减少缓存重新映射）和**健壮性**（不会导致雪崩效应）。

#### 一致性Hash算法的应用

在Memcached、Key-Value Store、Bittorrent DHT、LVS中都采用了Consistent Hashing算法，可以说Consistent Hashing 是分布式系统负载均衡的首选算法。

一致性hash算法是分布式中一个常用且好用的分片算法、主要使用于分布式缓存系统，或者数据库分库分表算法。在guava 和java的API中都有一致性hash的工具类，我们可以直接的使用。现在的互联网服务架构中，为避免单点故障、提升处理效率、横向扩展等原因，分布式系统已经成为了居家旅行必备的部署模式。

#### JAVA的简单实现一致性Hash算法

带有虚拟节点的代码

用于计算和建立虚拟节点关系类

```java
public class ConsistentHash {

    //虚拟节点
    private SortedMap<Integer, Node> virNode;

    //真实的节点,这里使用LinkedList更好
    private List<Node> realNode;

    private int nodeNum ;

    private final int DEFAULT_NODE_NUM = 10; // 每个机器节点关联的虚拟节点个数

    //初始化一直性Hash对象，向里面添加真实的节点
    public ConsistentHash(List<Node> realNode) {
        this.realNode = realNode;
    }

    public ConsistentHash(List<Node> realNode,int nodeNum) {
        this.realNode = realNode;
        this.nodeNum=nodeNum;
    }

    //初始化一致性Hash对象
    public void init() {
        virNode = new TreeMap<>();
        realNode.stream().forEach(node->dealNode(node,DEFAULT_NODE_NUM));  // 每个真实机器节点都需要关联虚拟节点
    }

    //初始化一致性Hash对象
    public void initNum() {
        virNode = new TreeMap<>();
        realNode.stream().forEach(node->dealNode(node,nodeNum)); // 每个真实机器节点都需要关联虚拟节点
    }

    //在每一个真实创建 DEFAULT_NODE_NUM个虚拟节点
    private void dealNode(Node realNode,int num) {
        for (int n = 0; n < num; n++) {
            // 一个真实机器节点关联NODE_NUM个虚拟节点
          virNode.put(getHash(realNode.getIPAddress()+"-"+realNode.getName()+"-"+n), realNode);
          System.out.println("虚拟节点-"+n+"-被加入到真实节点-"+realNode.getName()+"-hashCode:"+getHash(realNode.getIPAddress()+"-"+realNode.getName()+"-"+n));
        }
    }
    //根据Key去查找真实节点信息
    public Node getRealNode(String key) {
        SortedMap<Integer, Node> tail = virNode.tailMap(getHash(key)); // 沿环的顺时针找到一个虚拟节点
        if (tail.size() == 0) {
            return virNode.get(virNode.firstKey());
        }
        return tail.get(tail.firstKey()); // 返回该虚拟节点对应的真实机器节点的信息
    }
    //获取Hash值
    public static Integer getHash(String str) {
        return FNVHash1(str);
    }
    /**
     *使用FNV1_32_HASH算法计算服务器的Hash值,这里不使用重写hashCode的方法，最终效果没区别
     */
    public static int FNVHash1(String data) {
        final int p = 16777619;
        int hash = (int) 2166136261L;
        for (int i = 0; i < data.length(); i++)
            hash = (hash ^ data.charAt(i)) * p;
        hash += hash << 13;
        hash ^= hash >> 7;
        hash += hash << 3;
        hash ^= hash >> 17;
        hash += hash << 5;
        // 如果算出来的值为负数则取其绝对值
        if (hash < 0) hash = Math.abs(hash);
        return hash;
    }

}
```

用于存放数据的Node类

```java
/**
 *  用于插入数据的节点
 */
public class Node {

    private String name; //名字

    private String IPAddress;//IP地址

    private String Desc;//描述

    private List<String> data=new ArrayList<>();

    public Node(String name, String IPAddress, String desc) {
        this.name = name;
        this.IPAddress = IPAddress;
        Desc = desc;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getIPAddress() {
        return IPAddress;
    }

    public void setIPAddress(String IPAddress) {
        this.IPAddress = IPAddress;
    }

    public String getDesc() {
        return Desc;
    }

    public void setDesc(String desc) {
        Desc = desc;
    }

    public List<String> getData() {
        return data;
    }

    public void setData(List<String> data) {
        this.data = data;
    }
}

```

用于测试的client类

```java
public class ConsistentClient {
    public static void main(String[] args) {
        Node node1=new Node("node1","192.168.0.1","nice");
        Node node2=new Node("node2","192.168.0.2","bad");
        Node node3=new Node("node3","192.168.0.3","good");
        Node node4=new Node("node4","192.168.0.4","great");
        Node node5=new Node("node5","192.168.0.5","wonderful");
        List<Node> readList=new  LinkedList();
        readList.add(node1);
        readList.add(node2);
        readList.add(node3);
        readList.add(node4);
        readList.add(node5);
        ConsistentHash consistent=new ConsistentHash(readList);
        consistent.init();
        List<String> keys=new ArrayList<>();
        for(int i=0;i< 100;i++){
            keys.add("taiyan"+i);
        }

        for(int i=0; i<keys.size(); i++) {
            consistent.getRealNode(keys.get(i)).getData().add(keys.get(i));
            System.out.println("[" + keys.get(i) + "]的hash值为" + consistent.getHash(keys.get(i)) + ", 被路由到结点[" + consistent.getRealNode(keys.get(i)).getIPAddress() + "]");
        }

        //打印每个节点的数据量
        for(Node node:readList){
            System.out.println("Node"+node.getIPAddress()+"的数据"+node.getData().size());
        }
    }
}
```

运行结果

```
Node192.168.0.1的数据1093
Node192.168.0.2的数据2411
Node192.168.0.3的数据2207
Node192.168.0.4的数据1458
Node192.168.0.5的数据2831
```

当一个节失效后运行结果

```
Node192.168.0.1的数据1093
Node192.168.0.2的数据2625
Node192.168.0.3的数据2675
Node192.168.0.4的数据3607
```

其实可以对比上面能看出来，节点1的数据没有变化，其中节点2，3，4原来的数据是有增加，而不是集中到一个节点。同时这次数据变动也就只变动了2831个，其他的数据不受任何影响。没有虚拟节点我没有写，但是你可以在ConsistentHash类中删除掉虚拟节点的操作即可。这里也不多的说明

#### 总结

常用的算法是对hash结果取余数 (hash() mod N)：对机器编号从0到N-1，按照自定义的hash()算法，对每个请求的hash()值按N取模，得到余数i，然后将请求分发到编号为i的机器。但这样的算法方法存在致命问题，如果某一台机器宕机，那么应该落在该机器的请求就无法得到正确的处理，这时需要将当掉的服务器从算法从去除，此时候会有(N-1)/N的服务器的缓存数据需要重新进行计算；如果新增一台机器，会有N /(N+1)的服务器的缓存数据需要进行重新计算。对于系统而言，这通常是不可接受的颠簸（因为这意味着大量缓存的失效或者数据需要转移）。

典型的应用场景是： 有N台服务器提供缓存服务，需要对服务器进行负载均衡，将请求平均分发到每台服务器上，每台机器负责1/N的服务。

一致hash算法，解决了Hash取余算法中，单调性的问题，同时增加虚拟节点的方法，解决了负载的问题，使得数据在机器增加或减少时缓存有更高的利用率。

但是一致hash算法也有不好的地方，就是数据迁移问题。因为所有数据都分散到各个机器上，没有很好的聚合，所以对于数据迁移是有一定困难。

所以没有银弹，每一种算法只是解决一种相对应的问题。不要手里拿着锤子，就看什么都是钉子，还是要具体问题具体分析。

#### 杂记

首先来说假如key（关键字）是随机分布的，那么无所谓一定要模质数。但在实际中往往关键字有某种规律，例如大量的等差数列，那么公差和模数不互质的时候发生碰撞的概率会变大，而用质数就可以很大程度上回避这个问题。质数并不是唯一的准则。

[一个关于是否要取质数的博客](http://www.vvbin.com/?p=376)

计算Hash的几种方法

- 直接取余法： f(x)=(x+MAX)mod size 其中MAX是一个比较大的数，size是存储空间
- 平方取中法：先计算关键字的平方然后有目的的选取中间的若干位作为散列地址。若，假设散列地址范围是[000,999],k=4731则k^2=22382361取第三位至第五位作为其散列地址，则H(k)=382
- 叠加法：把关键字分割成位数相同的几部分，如最后一部分位数不够，则左边可以空缺。
- 基数转换法：设原关键字是十进制数，人为的当做q进制数，再转换成十进制数作为散列地址。
- 除留余数法：设散列范围的长度为m，将关键字值k除以某数p(p<=m)以后所得的余数作为散列地址。对应的散列函数为H(k)=k mod p

#### 参考

[一致性哈希算法](https://wizardforcel.gitbooks.io/the-art-of-programming-by-july/content/a.3.html)

[Hash算法大全](https://blog.csdn.net/l1028386804/article/details/54573106default)

[一致性Hash算法和代码的实现](https://blog.csdn.net/u011305680/article/details/79721030)