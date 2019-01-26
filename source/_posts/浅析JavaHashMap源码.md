---
title: 浅析Java HashMap源码
date: 2018-03-05 09:21:08
tags: [java,HashMap]
categories: 源代码
---

##### 回顾

- 之前写过一篇关于[HashMap的文章](https://blog.51cloud.win/2017/05/14/HashMap/#more)，里面介绍了关于HashMap的基本概念，和简单的数据结构，但是并没有对HashMap的源码进行系统的分析。

- HashMap主要结构是一个数组，数组的下标是hash值取余之后数，存储key和value的信息。允许key和value是null。HashMap的数组的使用率小于0.75（默认的0.75,也可以设置）。一般默认情况下数组的大小是16，随着数据的添加当HashMap的容量达到threshold（16*0.75=12）时候会扩容（resize），扩容是将原来的数组扩大两倍（HashMap的数组大小一定是2的n次方）。同时HashMap也不是线程安全的，在并发情况下resize可能会出现死锁。

- HashMap除了数组之外还有一个是链表，链表是解决解决HashMap的中hashcode取模以后的碰撞问题，正常的hashcode的范围很大，碰撞的几率很小但数组没有那么大的空间，对内存占用太大；要在为hashcode取模，由于取余会导致hashcode碰撞，为避免之前数组的数据被覆盖。HashMap在数组后面添加了列表，来解决这个问题。在JDK8以后当链表长度超过8以后就会被替换成一个红黑树，这样可以提高查找和插入效率。

   ![](/img/hashMap/hashMap-01.png)

##### HashMap是如何设计的

HashMap是实现了Map接口、允许null键/值、非同步、不保证有序(比如插入的顺序)、也不保证序不随时间变化。在HashMap中有两个很重要的参数，容量(Capacity)和负载因子(Load factor)。Capacity就是buckets的数目，loadFactor就是哈希桶(就是数组)填满程度的最大比例。如果对迭代性能要求很高的话不要把Capacity设置过大，也不要把loadFactor设置过小。当哈希桶填充的数目（即HashMap中元素的个数）大于Capacity*loadFactor时就需要调整哈希桶的数目为当前的2倍也就是扩容，可能和上面的有些重复。

计算Hash值，根据Hash值计算哈希桶中数组的位置。由于HashMap使用数组+链表的方式实现，所以能否快速计算Hash值和根据Hash值找到元素的位置很重要。下面就是Hash值是如何计算的

```java
//方法一,JDK1.7和1.8
static final int hash(Object key) {   
     int h;
     //第一步 取hashCode值 h = key.hashCode() 
     //第二步 高位参与运算 h ^ (h >>> 16) ，减少hashCode的大小，计算hash值
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
//方法二：jdk1.7的源码，jdk1.8没有这个方法，但是实现原理一样的。没有单独抽出来
static int indexFor(int h, int length) {  
     return h & (length-1);  //第三步 取模运算，获取数组下标
}
```

这里的Hash算法本质上就是三步：取key的hashCode值、高位运算、取模运算。

对于任意给定的对象，只要它的hashCode()返回值相同，那么程序调用方法一所计算得到的Hash码值总是相同的。我们首先想到的就是把hash值对数组长度取模运算，这样一来，元素的分布相对来说是比较均匀的。但是，模运算的消耗还是比较大的，在HashMap中是这样做的：调用方法二来计算该对象应该保存在table数组的哪个索引处。

这个方法非常巧妙，它通过h & (table.length -1)来得到该对象的保存位（数组的下标），而HashMap底层数组的长度总是2的n次方，这是HashMap在速度上的优化。当length总是2的n次方时，**h& (length-1)运算等价于对length取模，也就是h%length，但是&比%具有更高的效率**。但是hash值也就与数组的大小相关，所以每次resize的时候要重新进行Hash。所以链表和树都会被改变，resize是一个不小的开销（for循环套do-while循环），而且多线程下会出现并发问题。

在JDK1.8的实现中，优化了高位运算的算法，通过**hashCode()的高16位异或低16位**实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。

![](/img/hashMap/hashMap哈希算法例图.png)

##### HashMap是如何实现的

首先看一下HashMap中的变量和他们的用途。

```java

	//HashMap默认初始化大小，大小为16。它一定的2的倍数，这里使用位运算实现的，1左移4位
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

  
	//HashMap容量的最大值2的30次方。如果想指定更高的值可以通过构造函数指定。容量是2的背时
    static final int MAXIMUM_CAPACITY = 1 << 30;

  
	//默认的负载因子，如果不在构造函数中指定，为了防止HashMap冲突，数组内不是全部使用，而是使用一部分。
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

	//从链表转换成红黑数的阈值，当链表长度长度为8同时MIN_TREEIFY_CAPACITY大于64时，就会将冲突的链表升级红黑树。
    static final int TREEIFY_THRESHOLD = 8;

	//将红黑树转化成链表的阈值。同时hashMap的容量应该小于TREEIFY_THRESHOLD（64）。
    static final int UNTREEIFY_THRESHOLD = 6;

  	// 树化的最小容量64，也就是当HashMap的容量小于64时不进行树化（将链表转换成红树）
    static final int MIN_TREEIFY_CAPACITY = 64;

	//哈希桶（Node的数组），用于存放链表。它会随着数据量的增加去扩容。它的大小一般是2的n次方，同时不参与序列化   （transient，表示不被序列化）
    transient Node<K,V>[] table;

	//entry是存放key和Value的基本对象，这个一个set，一般会在遍历时候用到。根据它获取keySet和values
    transient Set<Map.Entry<K,V>> entrySet;

    //哈希表中的大小，存放了多少数据
    transient int size;

	//这个HashMap被结构化修改的次数。结构化修改例如resize。这个参数一般是来用于实现HashMap 迭代器的fail-fast机制的，一般和并发相关。例如HashMap不能循环读取的时候插入数据。会抛出ConcurrentModificationException
    transient int modCount;

    //哈希表内元素数量的阈值，当哈希表内元素数量超过阈值时，会发生扩容resize()。threshold=
    int threshold;
```

HashMap的基本构造函数，根据HashMap的参数来构造hashMap。

```java

  // HashMap的构造函数，initialCapacity用于指定初始化容量，loadFactor用于指定负载因子，也就是数组的最大使用率
  public HashMap(int initialCapacity, float loadFactor) {
    	//如果初始化大小小于0，抛出非法参数异常异常（边界处理）
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
    	//如果最初始化的容量比默认的最大容量大，那就指定最大容量为初始化容量。否则最大容量是默认最大容量
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
    	// 判断负载因子是大于0，是小数（浮点数）	
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;//通过上面的判断后赋值
        this.threshold = tableSizeFor(initialCapacity);//根据初始化容量获取阈值，HashMap的数组大小。数组大小一定是2的倍数。
    }
  //指定初始化容量，使用默认的负载因子也就是0.75f
  public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
 //使用默认的初始化容量16。和默认的负载因子0.75f
 public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
//通过一个Map，生成一个Hashmap,使用默认的负载因子0.75f
 public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
 //根据期望容量cap，返回2的n次方形式的 哈希桶的实际容量 length。 返回值一般会>=cap 
    static final int tableSizeFor(int cap) {
      	//经过下面的 或 和位移 运算， n最终各位都是1。
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
      	//查看是否越界，例如把11111 + 1 变成 100000 。返回哈希桶的阈值
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
//Map加入表中。参数m是要加入的Map，参数evict.一般会在resize和构造HashMap的时候用到
 final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();//获取m的大小
        if (s > 0) { //如果s<0就不用操作了
            if (table == null) { // pre-size
                float ft = ((float)s / loadFactor) + 1.0F;//根据m的元素数量和当前表的加载因子，计算出阈值。就是threshold. s除以loadFactor就是，哈希表的新容量。
                //如果新容量小于最大容量，就是新容量，否则就是最大容量。
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)//t相当于initialCapacity
                    threshold = tableSizeFor(t);//根据t获取获取阈值
            }
            else if (s > threshold)//如果map的size大于生成的阈值，扩容
                resize(); 
          	//循环把Map中的Key和Value放到HashMap中去
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);
            }
        }
    }
```

HashMap中常用的内部类

```java
 
	//这个类是HashMap中的基本类。实现了Map的Entry，下面还有TreeNode类。Hash表（哈希桶）指定就是Node的数组，里面是一个单向链表。从Node的数据结构也能看出来。
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;//存放hashcode
        final K key; //存放Key值
        V value; //c存放Value值
        Node<K,V> next; //存放冲突后下一个node节点的地址。（这是个链表）
		//Node的构造函数
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
		//重写set方法，get方法，equals方法，hashCode方法。
        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }
		//获取hashcode，是根据Key和Value的hashcode值
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }
		//这个set方法是有返回值的，把之前的值返回出来了，而不是直接覆盖掉
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

HashMap中比较关键的方法

- resize方法是初始化哈希桶或者扩容哈希桶的大小，扩容一般就是加倍（doubles table size），如果是当前哈希桶是null,分配符合当前阈值的初始容量目标。否则，因为我们扩容成以前的两倍。同时把冲突的在链表或者树上的值取出来重新在放到哈希桶里index为原来位置+oldCap，这个过程叫rehash。 元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![](/img/hashMap/hashMap 1.8 哈希算法例图2.png)

- 图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。n为hash表的长度

![](/img/hashMap/hashMap 1.8 哈希算法例图1.png)

- Hash表的扩容的具体过程。

![](/img/hashMap/jdk1.8 hashMap扩容例图.png)

因此，我们在扩充HashMap的时候，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的哈希桶。

```java

	//返回对象是一个Node数组，也就是个哈希桶；
    final Node<K,V>[] resize() {
      	//把当前哈希桶设置成旧哈希桶。
        Node<K,V>[] oldTab = table;
      	//如果旧哈希桶（也就是当前的哈希桶）是null，长度为0，否则获取旧哈希桶的长度，也就是容量。
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;//获取旧哈希桶的阈值。也就是当前的哈希桶的阈值
        int newCap, newThr = 0;//设置新哈希桶的阈值和容量为0。
        if (oldCap > 0) {//如果旧哈希桶的容量大于0说明是扩容，否则是创建哈希桶
            if (oldCap >= MAXIMUM_CAPACITY) {//如果旧哈希桶的容量大于等于最大容量()
                threshold = Integer.MAX_VALUE; //把阈值设置成integer最大值
                return oldTab;//直接返回旧哈希桶，说明不能在扩容。已经到了最大限度
            }
          	//1、新哈希桶的容量等于旧哈希桶容量的2倍，因为是左移1为2进制。
          	//2、扩容后新哈希桶的容量是否小于最大的容量同时旧哈希的容量大于等于默认容量16。
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // 阈值也要double。
        }
      	//如果当前表是空的，但是有阈值。代表是初始化时指定了容量、阈值的情况
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;//那么新表的容量就等于旧的阈值
        else {               // zero initial threshold signifies using defaults
         	// 如果当前表是空的,默认初始化Hash表(新建的hashMap)。使用默认的16个和0.75f的负载因子
            newCap = DEFAULT_INITIAL_CAPACITY; 
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {//如果新的阈值是0，对应的是当前表是空的，但是有阈值的情况
            float ft = (float)newCap * loadFactor;//根据新表容量 和 加载因子 求出新的阈值
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);  //进行越界修复
        }
        threshold = newThr;//把新哈希桶阈值值给当前阈值
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; //根据新hash表的变量，创建一个新哈希桶
        table = newTab;  //当前哈希桶等于新哈希桶
      	//从旧的哈希桶里面迁移数据。
        if (oldTab != null) {//如果旧的hash表不为空，迁移数据
            for (int j = 0; j < oldCap; ++j) { //根据旧哈希桶的容量去循环。rehash操做
                Node<K,V> e; //临时获取数组中对象
                if ((e = oldTab[j]) != null) { //如果桶中的对象不为空，把对象赋值给e
                    oldTab[j] = null;//清空这个对象，等待GC
                    if (e.next == null)//如果当前对象没有next说明后面没有冲突的链表
                       //根据e之前的哈希值和新容量，来获取数组下标的值，并把e放进去
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode) 
                        //如果e的类是TreeNode的话，将e强转成TreeNode然后把值插入进去
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                //如果发生过哈希碰撞，节点数小于8个。则要根据链表上每个节点的哈希值，依次放入新哈希桶对应下标置。
                    else { // preserve order
                      	//原索引 Node链表的头和尾。
                        Node<K,V> loHead = null, loTail = null;
                      	//原索引+oldCap Node链表的头和尾。
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next; //用于获取链表下一个对象的临时变量。与while循环中的e=next实现循环链表
                        do { //for循环里面+do while循环效率很低
                            next = e.next; //链表的从头上面获取
                            if ((e.hash & oldCap) == 0) {//原索引
                                if (loTail == null) //如果没有到链表的尾部，那么设置链表头和尾是一个
                                    loHead = e; //那么e就是链表的头
                                else
                                    loTail.next = e;//在链表的尾部添加e。
                                loTail = e;//链表的尾部也设置成e，也就是最新加进来的e
                            } 
                            else { //原索引+oldCap
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);//e的next为null，链表只有一个值
                      	//如果原索引不为空，写到新哈希桶中原来的位置
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                      	//如果原索引+oldCap不为空，新哈希桶中原来的位置+oldCap
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
      	//返回新的哈希表
        return newTab;
    }
```

- putVal方法，向hash表里插入值。如果参数onlyIfAbsent是true，那么不会覆盖相同key的值value，evict用于LinkedHashMap，下面是putVal方法的执行流程图。

![](/img/hashMap/hashMap put方法执行流程图.png)

可以结合上面的图看一下对应的源码。

```java
  //put方法向hash桶放入key和value
  public V put(K key, V value) {
    	//调用putVal方法，这里会覆盖key相同的值
        return putVal(hash(key), key, value, false, true);
    }

  // 参数hash是索引，key就是Map的key，Value是Map的value，onlyIfAbsent是否覆盖。evict用于LinkedHashMap
  final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    	//tab Hash桶，p临时链表的节点，n为hash表的长度（数组长度），i临时的索引变量
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)//如果hash表是空，或者hash桶的长度为0。
            n = (tab = resize()).length;//通过resize()方法创建一个hash桶，并获取桶的长度赋值给n
        if ((p = tab[i = (n - 1) & hash]) == null)//计算index，也就是数组的值。如果没有值，则没发生冲突,同时获取原来tab数组对应节点上的Node对象
            tab[i] = newNode(hash, key, value, null);//直接插入数组中（哈希桶）
        else {//发生了冲突，即要插入数组下标位置已经有一个Node对象
            Node<K,V> e; K k;//临时Node对象和key值。同时用于循环链表
          	//p是原来节点对象，判断新增对象和原来对象是否一样，hash值是否相等，key值是否相等
            if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;// 如果相等,表明该key为当前节点的第一个,将原值设置为当前 e 对象
            else if (p instanceof TreeNode) //如果p节点是tree节点，用红黑树的方式。
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);//把值放入到树里面
            else {//如果是链表，为链表处理的方式，binCount指的是这个链表的长度
                for (int binCount = 0; ; ++binCount) {//这是个死循环，没有结束条件，只有下面的break跳出
                    if ((e = p.next) == null) {//判断p节点后面有没有其他节点，即第一个冲突的。同时e被赋值next对象
                        p.next = newNode(hash, key, value, null);//如果没有直插入新节点
                        if (binCount >= TREEIFY_THRESHOLD - 1) // 判断hash列表是否需要树化，如果大于7也就是8个，从0开始算
                            treeifyBin(tab, hash);//将在tab中，同样hash值的对象树化。for循环套do-while循环
                        break;//跳出循环
                    }
                   // 如果链表中存在该 key（key插入成功了）,因为已经将该节点赋值给 e,所以直接结束循环。等待下面的方法对值进行更新
                    if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                  	//循环链表
                    p = e;
                }
            }
          	//如果e不等null
            if (e != null) { // existing mapping for key
                V oldValue = e.value;//把老的值取出来。
                if (!onlyIfAbsent || oldValue == null)//onlyIfAbsent为false，覆盖相桶的key的value
                    e.value = value;//覆盖value
                afterNodeAccess(e);//在访问之后的hook函数，在LinkedHashMap中会用到。
                return oldValue; //返回旧的值
            }
        }
        ++modCount;//修改的次数加1。
        if (++size > threshold)
            resize();//把size与阈值对比，如果大于resize扩容；
        afterNodeInsertion(evict);//用于LikedHashMap的hook函数。
        return null;
    }

 	//创建一个非树结构的节点
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
        return new Node<>(hash, key, value, next);
    }

   //树化哈希桶里面的节点，根据hash值
   final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;//是hash桶大小，index是数组下标，e是临时节点
     	//如果hash表是null（初始化哈希桶）或者tab的长度，小于最小树化的长度，就将扩容
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();//只是扩容，什么都不做
        else if ((e = tab[index = (n - 1) & hash]) != null) {//根据hash值，来获取链表的头节点。
            TreeNode<K,V> hd = null, tl = null; //创建红黑树节点
            do {
              	//将Node节点替换成TreeNode的节点，即红黑树的属性
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null) //如果尾部是null
                    hd = p;//头就是p（hd==head）
                else {
                    p.prev = tl;//设置设计p节点的前驱为之前的tail。treeNode节点继承LinkedHashMap.Entry。是双向的
                    tl.next = p;//尾部的next为p，也就是p是尾部。
                }
                tl = p;//设置尾部为p（tl==tail）
            } while ((e = e.next) != null); //直到链表的next为空。同时把next值赋给e
            if ((tab[index] = hd) != null)//如果hash表中第一个元素不为空
                hd.treeify(tab);//把tab树化
        }
    }

	//根据Node节点创建一个TreeNode节点。
    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        return new TreeNode<>(p.hash, p.key, p.value, next);
    }
```

- getNode方法。根据key的hash值和key来获取元素（对象）。在HashMap中通过hash值可以快速定位到元素所在的位置，然后根据key(key是唯一的)来找出元素所在的链表位置（或者树的位置）。所以从某个角度讲HashMap也是一个空间换时间的一种数据结构。他通过存储key和key的Hash值来快速定位元素。

```java
  //根绝Key获取value
  public V get(Object key) {
        Node<K,V> e;
    	//如果是null。返回null，如果有值返回值
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
//根据hash值和key获取存放数据的节点
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;//tab哈希桶，first 第一节点，n哈希桶的长度，k是key值
  		//哈希桶不等于空，哈希痛的长度大于0，隔绝hash值获取的第一个节点不为空 
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
          	//如果第一个key和哈希值就给定的相配，直接返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
          	//如果第一个对象，后面有值。
            if ((e = first.next) != null) {
              	//如果是红黑树节点，使用红黑树的获取办法
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {//不是的话，就循环链表来取出值，这里面用的do-while循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k)))) //如果hash值和key值都相等返回Node节点
                        return e;
                } while ((e = e.next) != null);//循环直到e的next节点为null
            }
        }
  		//否则返回null
        return null;
    }
```

- removeNode，删除哈希桶里面的值。同时要判断是否要使用值匹配。但是默认的都是不匹配的，如果使用匹配找到key和value相同的才会删除，否则返回null；

```java
 //删除一个元素节点，根据key值,返回元素节点的值
  public V remove(Object key) {
        Node<K,V> e;
    	//调用移除节点方法
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
//，根据Hash值，key，value，matchValue是否要匹配value值否则忽略，在删除节点后是否移动，如果不移动为false。
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
  		//声明使用的变量，如临时节点p，p节点一般都是哈希桶里的第一节点，哈希表的总长度n和下标index
        Node<K,V>[] tab; Node<K,V> p; int n, index;
  		//哈希桶不为空，长度大于0，根绝hash值获取节点不为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;//节点node和e，key和value
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p; //如果key值和hash值相等不为null，就把赋值node
            else if ((e = p.next) != null) {//否者循环链表,或者是红黑树
                if (p instanceof TreeNode)//获取树里面的节点
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {//获取链表里的节点
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
          	//找到元素后删除，和链表和树的移动。1首先node节点不为null，同时判断value节点值问题。
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)//如果是Tree节点，那么使用tree的操作,移除节点
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)//如果是第一个节点。直接替换。
                    tab[index] = node.next;
                else
                    p.next = node.next;//设置node的next节点
                ++modCount;//操作数+1
                --size;//大小-1
                afterNodeRemoval(node);//hook 方法
                return node;//返回node节点
            }
        }
        return null;
    }
//清空HashMap里面的值
public void clear() {
        Node<K,V>[] tab;
        modCount++; //操作数+1
        if ((tab = table) != null && size > 0) { //如果哈希表不为空，循环取出哈希表里面的值
            size = 0;//设置HashMap大小为0
            for (int i = 0; i < tab.length; ++i)
                tab[i] = null;//清空哈希桶里的数据，也就是清空数组
        }
    }
```

##### HashMap中线程安全问题

```java
	//hashMap中通过迭代器来获取，Map中的数据。通过迭代器前modCount与循环后modCount对比，如果不一样，抛出
	//ConcurrentModificationException异常。来实现fast-fail机制
	//迭代器，用于循环取出HashMap中的数据
    abstract class HashIterator {
        Node<K,V> next;        // next entry to return ，下一个entry
        Node<K,V> current;     // current entry，当前entry
        int expectedModCount;  // for fast-fail，用于fast-fail的变量
        int index;             // current slot，当前所在哈希桶的位置（数组下标）
		
      	//默认的构造函数，构造HashMap的迭代器
        HashIterator() {
            expectedModCount = modCount;//期待的修改次数，使用当前hashMap的修改次数
            Node<K,V>[] t = table;//哈希桶
            current = next = null;//初始化当前节点和next节点
            index = 0;//数组下标从0开始
            if (t != null && size > 0) { // advance to first entry,如果哈希桶不为空，size大于0
              	//找到第一个entry，如果next不为空，跳出循环。并把元素赋值给next，等于null继续找
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }
		//时候还存在next
        public final boolean hasNext() {
            return next != null;
        }
		//获取下一个Next节点
        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)//如果当前的修改次数不等于之前修改。也就是在这个过程HashMap发生修改。这里实现了fast-fail
                throw new ConcurrentModificationException();//抛出异常
            if (e == null)//如果next节点为null，抛出没回这样元素的异常，一般在做循环的时候都会做判断
                throw new NoSuchElementException();
          		//这里赋值e，同时把next也赋值。
            if ((next = (current = e).next) == null && (t = table) != null) {
              	//循环找到下一个元素
                do {} while (index < t.length && (next = t[index++]) == null);
            }
          	//返回当前节点
            return e;
        }
		//移除当前节点
        public final void remove() {
            Node<K,V> p = current;
            if (p == null)//如果当前节点是null抛出异常
                throw new IllegalStateException();
            if (modCount != expectedModCount)//fast-fail机制
                throw new ConcurrentModificationException();
            current = null;
          	//根据key删除节点
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;//同时更新操作次数
        }
    }
```

在多线程使用场景中，应该尽量避免使用线程不安全的HashMap，而使用线程安全的ConcurrentHashMap。那么为什么说HashMap是线程不安全的，因为在数据增加的时候会导致resize(扩容)，会导致链表上数据发生变化，容易在多线程情况下是很容易造成链表回路。在get的情况下造成死循环。

```java
//死循环的例子
public class HashMapInfiniteLoop {  //基于jdk1.7

    private static HashMap<Integer,String> map = new HashMap<Integer,String>(2，0.75f);  
    public static void main(String[] args) {  
        map.put(5， "C");  

        new Thread("Thread1") {  
            public void run() {  
                map.put(7, "B");  
                System.out.println(map);  
            };  
        }.start();  
        new Thread("Thread2") {  
            public void run() {  
                map.put(3, "A);  
                System.out.println(map);  
            };  
        }.start();        
  
```

map初始化为一个长度为2的数组，loadFactor=0.75，threshold=2*0.75=1，也就是说当put第二个key的时候，map就需要进行resize。通过设置断点让线程1和线程2同时debug到transfer方法的首行。注意此时两个线程已经成功添加数据。放开thread1的断点至transfer方法的“Entry next = e.next;” 这一行；然后放开线程2的的断点，让线程2进行resize。结果如下图

![](/img/hashMap/jdk1.7 hashMap死循环例图1.png)

注意，Thread1的 e 指向了key(3)，而next指向了key(7)，其在线程二rehash后，指向了线程二重组后的链表。线程一被调度回来执行，先是执行 newTalbe[i] = e， 然后是e = next，导致了e指向了key(7)，而下一次循环的next = e.next导致了next指向了key(3)。

![](/img/hashMap/jdk1.7 hashMap死循环例图2.png)

e.next = newTable[i] 导致 key(3).next 指向了 key(7)。注意：此时的key(7).next 已经指向了key(3)， 环形链表就这样出现了

![](/img/hashMap/jdk1.7 hashMap死循环例图4.png)

于是，当我们用线程一调用map.get(11)时，悲剧就出现了——Infinite Loop(无限循环)。

##### HashMap与HashTable 的区别

HashTable是线程安全的，且不允许key、value是null。HashTable默认容量是11。

HashTable是直接使用key的hashCode(key.hashCode())作为hash值，不像HashMap内部使用static final int hash(Object key)获取hash值。

HashTable取哈希桶下标是直接用模运算%.（因为其默认容量也不是2的n次方。所以也无法用位运算替代模运算）

扩容时，新容量是原来的2倍+1。int newCapacity = (oldCapacity << 1) + 1;

Hashtable是Dictionary的子类同时也实现了Map接口，HashMap是Map接口的一个实现类；

##### 参考

[HashMap源码解析（JDK8）](http://blog.csdn.net/zxt0601/article/details/77413921)

[Java 8系列之重新认识HashMap](https://tech.meituan.com/java-hashmap.html)

