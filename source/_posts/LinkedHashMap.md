---
title: LinkedHashMap
date: 2017-09-17 10:55:59
tags: [编程,java,源代码]
categories: java
---

- Linked是什么

  linked是串联，连接的意思。编程里面理解成是用**链表**实现的数据结构，一般与HashMap，List，Set什么联合使用，说明这些集合类，都是通过链表实现的，这些集合存储额都是有顺序的，按照放进去的顺序。

- 什么是链表，链表与数组的去别

  链表这个名称，估计学过数据结构的人都不陌生，**链表（Linked list）是一种常见的基础数据结构，是一种线性表，但是并不会按线性的顺序存储数据**。**也就是说他存储的信息的地址不是顺序的，与之相反的是数组。他的存储的信息和内存地址是连续的。**使用链表的好处就在于每次修改删除比较方便，你只要找到对于位置，修改掉前驱元素和后继元素的指针就可以，不用移动其他数据，而数组在新增和删除的时候要移动大量的数据。所以在查询方面数组有很的优势，而在频繁的修改的数据，使用链表速度快一点**。数组利用下标定位，时间复杂度为O(1)，链表定位元素时间复杂度O(n)； 数组插入或删除元素的时间复杂度O(n)，链表的时间复杂度O(1)**。同时链表也增加了存储空间，原来只存data，现在要存data的before和after。

- LinkedHashMap是什么

  LinkedHashMap是HashMap的子类，他实现了Map接口

  其扩展了 HashMap 增加了双向链表的实现。相较于 HashMap 的迭代器中混乱的访问顺序，LinkedHashMap 可以提供可以预测的迭代访问，**即按照插入序 (insertion-order) 或访问序 (access-order) 来对哈希表中的元素进行迭代**。从类声明中可以看到，LinkedHashMap 确实是继承了 HashMap，因而 HashMap 中的一些基本操作，如哈希计算、扩容、查找等，在 LinkedHashMap 中都和父类 HashMap 是一致的。插入序就是安装插入的顺序来访问，即从链表的头（frist）开始访问，如果是访问序，就是从链表的尾部（tail）来开始访问，即最新插入，或者是之前访问过的就是最先访问。但是，和 HashMap 有所区别的是，LinkedHashMap 支持按插入序 (insertion-order) 或访问序 (access-order) 来访问其中的元素。所谓插入顺序，就是 Entry 被添加到 Map 中的顺序，更新一个 Key 关联的 Value 并不会对插入顺序造成影响；而访问顺序则是对所有 Entry 按照最近访问 (least-recently) 到最远访问 (most-recently) 进行排序，读写都会影响到访问顺序，但是对迭代器 (entrySet(), keySet(), values()) 的访问不会影响到访问顺序。访问序的特性使得可以很容易通过 LinkedHashMap 来实现一个 LRU(least-recently-used) Cache，后面会给出一个简单的例子。之所以 LinkedHashMap 能够支持插入序或访问序的遍历，是因为 LinkedHashMap 在 HashMap 的基础上增加了双向链表的实现。下面是代码分析

  ```java
  public class LinkedHashMap<K,V>
      extends HashMap<K,V>
      implements Map<K,V>
  {
   //一个匿名内部类，用于存放相关节点信息，继承HashMap的Node内部类
   //多了Entry<K,V>类型的 before和after。类似组合模式
  static class Entry<K,V> extends HashMap.Node<K,V> {
          Entry<K,V> before, after;
          Entry(int hash, K key, V value, Node<K,V> next) {
              super(hash, key, value, next);
          }
      }

  //当前链表节点的头节点，最老的节点
  transient LinkedHashMap.Entry<K,V> head;
  //当前链表的尾部节点，最新的节点
  transient LinkedHashMap.Entry<K,V> tail;  
  }
  //将新节点 p 链接到双向链表的末尾
  //一个私有方法，把一个节点加到另一个节点后面，如果前驱节点为null，只有他一个节点，即他是head也是tail
   private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
          LinkedHashMap.Entry<K,V> last = tail;
          tail = p;
          if (last == null)
              head = p;
          else {
           //之前最后的节点是last，把last节点放到最新节点p的before，把last节点after设置成最新节点p
              p.before = last;
              last.after = p;
          }
      }
  //私有方法，把src链接到dst上，就是用dst替换src在双向链表中的位置
  private void transferLinks(LinkedHashMap.Entry<K,V> src,
                                 LinkedHashMap.Entry<K,V> dst) {
    	 	//将之前src的前驱，和后继都复制给dst，同时赋值给a和b，b是前驱，a是后继
          LinkedHashMap.Entry<K,V> b = dst.before = src.before;
          LinkedHashMap.Entry<K,V> a = dst.after = src.after;
    		//如果b为空，src原来就是head节点，把dst设置成head节点
          if (b == null)
              head = dst;
          else
           //否则把src的前驱节点的after设置成dst
              b.after = dst;
    		// 如果a为空,src是原来的tail节点，把dst设置成tail节点
          if (a == null)
              tail = dst;
          else
            //否则把src的后继节点before设置成dst
              a.before = dst;
      }
  //调用父类的重新初始化方法，把值设成null
  void reinitialize() {
          super.reinitialize();
          head = tail = null;
      }
  //创建一个新的entry节点，重写父类hashmap的方法
  Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
          LinkedHashMap.Entry<K,V> p =
              new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    		//将创建的新节点，加到列表后面
          linkNodeLast(p);
          return p;
      }
  //将TreeNode节点转换成普通节点。TreeNode节点是个红黑树，在hashMap中，当链表长度超过8时，会把entry链表转换
  //成TreeNode.普通节点即entry节点，是个单链表由key，value，next，hash组成
  Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
          LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
          LinkedHashMap.Entry<K,V> t =
              new LinkedHashMap.Entry<K,V>(q.hash, q.key, q.value, next);
   		//替换节点的方法
          transferLinks(q, t);
          return t;
      }
  //创建一个treeNode节点，加入到链表的最后，treeNode是hashmap的内部类，LinkedHashMap继承了HashMap
  TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
          TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
          linkNodeLast(p);
          return p;
      }
  //将一个entry节点替换成TreeNode节点，一般碰撞超过8个会用一个树来代替上面的链表
   TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
          LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
     		//新建一个treeNode节点
          TreeNode<K,V> t = new TreeNode<K,V>(q.hash, q.key, q.value, next);
     		//替换节点
          transferLinks(q, t);
          return t;
      }

  //移除节点的回调函数，这个函数在hashmap中声明，但是没有实现，然后在linkedhashmap实现。
  void afterNodeRemoval(Node<K,V> e) { // unlink
      //移除一个节点，双向链表中的连接关系也要调整，先将节点里的值取出来
      LinkedHashMap.Entry<K,V> p =
          (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    	//移除节点
      p.before = p.after = null;
      if (b == null)
          head = a;
      else
          b.after = a;
      if (a == null)
          tail = b;
      else
          a.before = b;
  }
  /*插入节点的回调函数，也是在hashmap中声明，在Linkedhashmap中实现，evict（赶出），在hashmap照片您好evict都是等于true,本函数也就是在没有指定removeEldestEntry这个函数等于true的时候，是不会移除第一个节点，都只是向后插入，而不替换之前的节点，但是，如果是set的话，*/
   void afterNodeInsertion(boolean evict) { // possibly remove eldest
          LinkedHashMap.Entry<K,V> first;
     		//对是否删除eldest节点做判断
     		//如果evict是true，节点不为空，removeEldestEntry函数返回的是true，删除eldest节点
     		//一般默认removeEldestEntry函数返回的是FALSE，在LinkedHashMap，所以只是把head赋值给了first，并
     		//没有移除eldest节点，如果你要设置LRU算法的时候覆写该方法，一般的实现是，当设定的内存
    		//（这里指节点个数）达到最大值时，返回true
          if (evict && (first = head) != null && removeEldestEntry(first)) {
              K key = first.key;
            	//hashMap中的方法，用来节点，linkedhashMap继承过来的
              removeNode(hash(key), key, null, false, true);
          }
      }
  //访问节点的回调函数，这里实现了访问序和插入序的实现
    void afterNodeAccess(Node<K,V> e) { // move node to last
          LinkedHashMap.Entry<K,V> last;
      	//如果是访问序，把当前节点放到tail节点
          if (accessOrder && (last = tail) != e) {
              LinkedHashMap.Entry<K,V> p =
                  (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
              p.after = null;
            	//如果当前的节点的前驱点是空，也可以说，当前节点为head结点那，把after节点，放到head上，
            	//如果不是把 p 的前驱节点节点的后继节点改成 p 的后继节点。
              if (b == null)
                  head = a;
              else
                  b.after = a;
           //如果p的后继节点不为空，p的后继节点与的前驱节点改成p的前驱节点
          //如果不是也就是p节点是没有后继节点，也就是tail节点那把p的前驱节点复制给last节点，last节点是tail节点
              if (a != null)
                  a.before = b;
              else
                  last = b;
            	//如果last节点为空，也就是只有一个节点（当前的tail节点），把当前节点赋值给head节点
            	//如果不是last节点不为空，把当前节点的前驱设置成last节点，把last节点的后继设成当前节点。
              if (last == null)
                  head = p;
              else {
                  p.before = last;
                  last.after = p;
              }
            	//把当前节点放到tail节点上
              tail = p;
              ++modCount;
          }
      }

  	//遍历LinkedHashMap，将LinkedHashMap中的key和value实现序列化
      void internalWriteEntries(java.io.ObjectOutputStream s) throws IOException {
          for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
              s.writeObject(e.key);
              s.writeObject(e.value);
          }
      }

  //遍历LinkedHashMap，查找是否包含某个值
     public boolean containsValue(Object value) {
          for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
              V v = e.value;
              if (v == value || (value != null && value.equals(v)))
                  return true;
          }
          return false;
      }
  //获取value值，getNode()方法，实现在hashMap类中
    public V get(Object key) {
          Node<K,V> e;
          if ((e = getNode(hash(key), key)) == null)
              return null;
          if (accessOrder)
              afterNodeAccess(e);
          return e.value;
      }
  //是否移除最老的entry，记录，默认是false，如果写LRU缓存可以重写这方法。
  protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
          return false;
      }
  //最后说一下LinkedHashMap的构造函数
  //指定LinkedHashMap的初始大小，和负载因子，防止hash碰撞过多，这里是符合泊松分布的。默认加载因子是0.75，
  //super这里调用的是HashMap的构造方法。
  //访问序（accessOrder是false）。
    public LinkedHashMap(int initialCapacity, float loadFactor) {
          super(initialCapacity, loadFactor);
          accessOrder = false;
      }

  //这个应该是他最全的构造函数，这里accessOrder是指是否访问序
  public LinkedHashMap(int initialCapacity,
                           float loadFactor,
                           boolean accessOrder) {
          super(initialCapacity, loadFactor);
          this.accessOrder = accessOrder;
      }
  ```


其实LinkedHashMap中还有几个内部类没有说，但是我在这里就不多解释，他都是重写或者实现了上层的方法。所以我这也就不过多的介绍，本文是基于JDK 1.8.0_121的进行的分析。LinkedHashMap的好处就是插入和删除比较快，他不会像数组那样每次删除增加都会移动，但是，每次查询都会比较慢，毕竟是连续的存储，只有知道当前节点才能知道下一个节点。不如数组快。同时采用链表的设计会对内存的要求增大(每个节点不仅存数据，还要存前驱后继的地址)，不LinkedHashMap是有序的，在很多要求有序的场景下可以使用。

最后还要LRU缓存的实现，这个是从网上找的例子。

```java
//http://blog.jrwang.me/2016/java-collections-linkedhashmap/ 代码出处
public class CacheImpl<K,V> {
    private Map<K, V> cache;
    private int capacity;

    public enum POLICY {
        LRU, FIFO
    }

    public CacheImpl(int cap, POLICY policy) {
        this.capacity = cap;
        cache = new LinkedHashMap<K, V>(cap, 0.75f, policy.equals(POLICY.LRU)){
            //超出容量就删除最老的值
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > capacity;
            }
        };
    }

    public V get(K key) {
        if (cache.containsKey(key)) {
            return cache.get(key);
        }
        return null;
    }

    public void set(K key, V val) {
        cache.put(key, val);
    }

    public void printKV() {
        System.out.println("key value in cache");
        for (Map.Entry<K,V> entry : cache.entrySet()) {
            System.out.println(entry.getKey() + ":" + entry.getValue());
        }
    }

    public static void main(String[] args) {
        CacheImpl<Integer, String> cache = new CacheImpl(5, POLICY.LRU);

        cache.set(1, "first");
        cache.set(2, "second");
        cache.set(3, "third");
        cache.set(4, "fourth");
        cache.set(5, "fifth");
        cache.printKV();

        cache.get(1);
        cache.get(2);
        cache.printKV();

        cache.set(6, "sixth");
        cache.printKV();
    }
}
```

