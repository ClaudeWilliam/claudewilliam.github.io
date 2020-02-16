---
title: 浅谈volatile
date: 2020-02-16 09:21:08
tags: [java,多线程]
categories: java
---

#### 前言

通过上一篇文章[浅谈JMM](https://blog.51cloud.win/2020/01/28/%E6%B5%85%E8%B0%88JMM/#more)，我们知道现代计算由于多级缓存和多CPU的问题，导致多线程下会出现可见性和竞争性的问题。Java为了解决这两个问题，使用了两个关键字一个是volatile解决可见性问题，一个是 synchronized 解决同步问题（synchronized也解决可见性的问题）。它们的具体实现都是由`JMM`定义的8种原子从去实现的，下面我们就了解一下`JMM`中的8种原子操作。

#### JMM中的8种原子操作

之前的一篇文章由讲到，由于各个工作内存时隔离的，只有通过主内存才能实现Java进程中间数据的传输，所以就会出现可见性问题和竞争性问题。为了解决问题，`JMM`抽象出主内存和工作内存之间具体的协议（变量如何从主内存copy到工作内存，工作内存如何同步回主内存的实现细节）。Java 内存模型定义了8种操作，Java虚拟机实现时必须保证这8种操作时原子的，不可分割的（对于double和long这种在某些操作是由例外的）；后面最新`JSR-133`已经把这8种操作简化成4种，但是原理没变我们后面在去解释。

- lock（锁定）：作用于主内存变量，它把一个变量标识为一条线程独占的状态。
- unlock（解锁）：作用于主内存的变量，它把一个处于锁定的状态的变量释放出来，释放后才可以被其他线程锁定。

lock+unlock主要作用主内存，使得主内存的变量保持锁定状态。

- read（读取）：作用于主内存变量，它把一个变量的值从主内存传输到线程工作内存种，以便后面的的load动作的使用。（可以理解为半步操作）
- load（载入）：作用于工作内存变量，它把read操作从主内存种得到变量值放到工作内存的变量副本中。（半步操作）

read+load的操作菜把变量真正的放到工作内存种使用，只有load后CPU工作内存才会有这个变量。

- use（使用）：作用于工作内存的变量，它把工作内存中的一个变量值传递给执行引擎，每当虚拟机遇到一个需要使用变量的值的字节码指令时都会执行这个操作。（工作内存中的值被CPU给使用）
- assign（赋值）：作用于工作内存变量，它把执行引擎的值赋值给工作内存的变量，每当虚拟机遇到一个赋值的字节码指令都会执行这个操作。

use+assign就是把将工作内存中的值给CPU计算和计算完之后的赋值给工作内存的操作。

- store（存储）：作用于工作内存变量，它把工作内存的一个变量值传输给主内存中，以便后面的write操作使用。（半步操作）
- write（写入）：作用于主内存变量，它把store从存储的工作内存变量值（store后已经在主内存）存储在主内存中。（半步操作）

store+write主要就是把工作内存中的变量同步到主内存中。

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/JMM.png)

上述的8种内存交互操作，都是原子性和不可分割的。我们能从上面看出来其实它read+load和store+write可以合并成一个操作，但是起初`JMM`把他们定义成两个，这样可以提高系统的吞吐率，批量的load/write操作可以提高系统 效率(个人猜测)，同时也说明上面不可分割的原则。那这8种操作也不是随便使用的，他们也有他们的使用规则。下面这个规则和重要，我们可以根据这个规则猜想到新的四种操作是把那些操作结合在一起。因为他是可以放到一起，保证内存交互正确。

- 不允许read和load、store和write操作之一单独出现，即不允许一个变量从主内存读取到工作内存不接受，或者工作内存发起回写但是主内存不接受情况。
- 不允许一个线程丢弃它最近的assign操作，即变量在工作内存种改变胡必须把变化同步给主内存。
- 一个新的变量只能在主内存中诞生，不允许在工作内存中直接使用一个未被初始化（load/或者assign）的变量。换句话说就是对一个变量实施use、store操作之前，必须执行assign和load操作
- 一个变量在同一个时刻只允许线程对其进行lock操作，但lock操作可以被一个线程重复执行多次（可重入锁是不是有点像），多次执行lock后，只有执行相同unlock操作变量才会被解锁。
- 如果对一个变量执行lock操作，那将会清空工作内存中变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作以初始化变量的值。
- 如果一个变量没有被lock操作锁定，那就不允许对它执行unlock操作，也不允许unlock一个被其他线程锁定的变量。（unlock和lock也是成对出现的）
- 对一个变量执行unlock操作之前，必须先把此变量同步回主内存中（执行store、write操作）；

如果把一个主内存copy到工作内存，那就要顺序执行read和load操作，如果把工作内存的变量同步到主内存，就要按顺序执行store和write操作。注意Java内存模型只要求两个操作必须按顺序执行，但是不要求连续执行，也就是read与load之间插入其他指令，例如对主内存a，b进行访问，一种可能出现顺序read a、read b、load b、load a等等。

从上面我们能看出来，这个内存交互的8个操作描述很严谨，但是很烦琐而且实现起来很麻烦。后面Java设计也意识这个问题，将Java内存模型操作简化成为read、write、lock、unlock四种，但这个语言描述的简化但是基础设计未改变。从上面我们可以推测一下这个8个对用4步操作对应关系。lock和unlock应该是直接对应没有改变，read等价于read+load+use 3步操作合成；write等价于assign+store+write 3步操作合成。

#### volatile的变量

volatile关键字是Java虚拟机提供的最轻量的同步机制。一个变量被声明为volatile它会保证两个特性，一个是可见性。这个可见性我们上面也提到过，就是当一个线程修改一个值的时候，其他线程就会立刻知道，改变后的新值对于其他线程是最新的。（read+load，store+write去实现，他保证了可见性）

volatile第二语义就是保证其有序性。现代处理器都是乱序执行的，这样可以提高吞吐量(**当某些指令不在指令缓存中时，需要从主存中加载指令到高速缓存中，这个时间对于CPU来说是很长的，乱序执行允许CPU执行后面的指令而不是等待当前IO，这种情况主要出现在跳转指令**)，只要保证执行结果和顺序执行的是一致的就OK。**同样编译器也会有优化，也会对指令进行重排序**。volatile还要保证代码逻辑的有序，这个时候就要使用内存屏障和禁止指令重排来实现有序性。

Java内存模型种对volatile变量的特殊规则，假定T表示一个线程，V和W分别是两个volatile变量，那么在进行read、load、use、assign、store和write操作要满足下面的规则。

- 只有当线程T对变量V执行的前一个动作是load的时候，线程T才能对变量V执行use操作；并且，当只有线程T对变量V的最后的动作是use的时候，线程才能对变量W执行load动作。线程T对变量V的use动作可以认为和线程T对变量V的read、load动作关联，必须连续且一起出现。（**这条规则要求在工作内存中，每次使用V都必须先从主内存刷新最新的值，用于保证能看见其他线程对变量V所做的修改**）
- 只有线程T对变量V执行的前一个动作是assign的时候，线程T才能对变量V执行store动作；并且，只有当线程T对变量V执行最后一个动作是store的时候，线程T才能对变量W执行assign动作。线程T对变量V的assign动作是和T对变量store、write动作是关联的，必须连续且一起出现。（**这条规则要求在工作内存中，每次修改V后都必须立刻同步回主内存，用于保证其他线程可以看到自己对变量V所做的修改**）
- 假定**动作A**是线程T对变量V使用use和assign动作，假定**动作F**是和**动作A**相关联的load或store动作，假定**动作P**是和**动作F**相关联对变量V的read或write动作。与此类似，假定**动作B**是线程T变量W使用use或者assign动作，假定**动作G**是和**动作B**相关联的load或store动作，假定**动作Q**是和**动作G**相应的对变量W的read和write动作。**如果动作A先于动作B，那么动作P先于动作Q**。（**这条规则要求volatile修饰的变量不会被指令重排优化保证了代码的执行顺序是相同的，即先load的先使用**）

上面这个例子是说明Java内存模型保证可见性和有序性那具体是怎么实现的时候我们就说一下。

#### 内存屏障和禁止重排编译优化

**内存屏障**（Memory barrier），也称**内存栅栏**，**内存栅障**，**屏障指令**等，是一类同步屏障指令，它使得 CPU 或编译器在对内存进行操作的时候, 严格按照一定的顺序来执行, 也就是说在memory barrier 之前的指令和memory barrier之后的指令不会由于系统优化等原因而导致乱序。内存屏障是CPU级别的指令不同的CPU架构（`x86`,`ARM`）可能是不一样的实现方式，这里就不详细展开了。下面说到的load和store都是值的CPU内存屏障的load和store的屏障不要混淆了。

- 完全内存屏障(full memory barrier)保障了早于屏障的内存读写操作的结果提交到内存之后，再执行晚于屏障的读写操作。
- 内存读屏障(read memory barrier)仅确保了内存读操作；
- 内存写屏障(write memory barrier)仅保证了内存写操作。

硬件层的内存屏障分为两种：Load Barrier 和 Store Barrier即读屏障和写屏障。

内存屏障有两个作用：阻止屏障两侧的指令重排序；强制把写缓冲区/高速缓存中的脏数据等写回主内存，让缓存中相应的数据失效。

- 对于Load Barrier来说，在指令前插入Load Barrier，可以让高速缓存中的数据失效，强制从新从主内存加载新数据；
- 对于Store Barrier来说，在指令后插入Store Barrier，能让写入缓存中的最新数据更新写入主内存，让其他线程可见。

由于内存屏障的作用，避免了volatile变量和其它指令重排序、线程之间实现了通信，使得volatile表现出了锁的特性。

在Java中对于volatile修饰的变量，编译器在生成字节码时，会在指令序列中插入**内存屏障**禁止处理器重排序

#### Java中的内存屏障

Java中的内存屏障分四种，`LoadLoad`、`StoreStore`、`StoreLoad`、`LoadStore`。主要是使用load和store指令来实现的。这里也是Java抽象出来的内存屏障为了使让不同的平台的硬件去适配，然后可以跨平台的使用。表中的`load1`、`load2`、`store1`和`store2`都是`JMM`定义的交互操作，在这些操作中间会有内存屏障保证有序性，也就是Happen before 原则。

| 屏障类型            | 指令示例                 | 说明                                                         |
| :------------------ | :----------------------- | :----------------------------------------------------------- |
| LoadLoad Barriers   | Load1;LoadLoad;Load2     | 该屏障确保Load1数据的装载先于Load2及其后所有装载指令的的操作。（**操作系统保证在Load2及后续的读操作读取之前，Load1已经读取**） |
| StoreStore Barriers | Store1;StoreStore;Store2 | 该屏障确保Store1立刻刷新数据到内存(使其对其他处理器可见)的操作先于Store2及其后所有存储指令的操作。（**操作系统保证在Store2及后续的写操作写入之前，Store1已经写入**） |
| LoadStore Barriers  | Load1;LoadStore;Store2   | 确保Load1的数据装载先于Store2及其后所有的存储指令刷新数据到内存的操作。（**操作系统保证在Store2及后续写入操作执行前，Load1已经读取**） |
| StoreLoad Barriers  | Store1;StoreLoad;Load2   | 该屏障确保Store1立刻刷新数据到内存的操作先于Load2及其后所有装载装载指令的操作。它会使该屏障之前的所有内存访问指令(存储指令和访问指令)完成之后,才执行该屏障之后的内存访问指令（**操作系统保证在Load2及后续读取操作执行前，Store1已经写入**） |

`StoreLoad` 同时具备其他三个屏障的效果，因此也称之为`全能屏障`，是目前大多数处理器所支持的；但是相对其他屏障，该屏障的开销相对昂贵。

##### volatile语义中的内存屏障

volatile的内存屏障策略非常严格保守，非常悲观且毫无安全感的心态。

在volatile写操作前插入`StoreStore`屏障（store1和store2指令不能交换顺序）；

在写操作后插入`StoreLoad`屏障（store1 和load2指令不能交换顺序）；

在每个volatile读操作前插入`LoadLoad`屏障（load1和load2指令不能交换顺序），

在读操作后插入`LoadStore`屏障（load1和store2不能交换顺序）；

我们来实战一下这个屏障具体使怎么样的，写上一个表格和一段代码，具体就可以分析了。

| （竖行操作在前，横行操作在后） | 普通读       | 普通写        | volatile读    | volatile写     |
| ------------------------------ | ------------ | ------------- | ------------- | -------------- |
| **普通读**                     |              |               | *LoadLoad*    | **LoadStore**  |
| **普通写**                     |              |               | *StoreLoad*   | **StoreStore** |
| **volatile读**                 | **LoadLoad** | **LoadStore** | **LoadLoad**  | **LoadStore**  |
| **volatile写**                 | *LoadStore*  | *StoreStore*  | **StoreLoad** | **StoreStore** |

我们看到普通读写之间使没有内存屏障的，只有和volatile变量相关才会有屏障。斜体的表示不存在也可以，那个没顺序的要求。黑色加粗部分使一定要有的，不然不保证顺序。

```java
class VolatileExample {
    int a = 0;
    volatile boolean flag = false;

    public void writer() {
        a = 1;                   //1 普通写
        //StoreStore，a对flag可见
        flag = true;               //2 volatile写
        //StoreLoad，flag和a对后续可见           
    }

    public void reader() {
        //LoadLoad，flag和a可见
        if (flag) {                //3volatile读
        //LoadStore，flag和a可见
            int i =  a;           //4普通变量写
        }
    }
}
```

##### 内存屏障的实现

查看编译后的代码其实能发现会多出一个`lock addl $0x0,(%esp)`操作，根据IA32手册知道，这个操作可以让前面的volatile变量的修改对其他处理器立即可见，该动作可以导致处理器的或者内核无效化其缓存。

在具体的执行上，**它先对总线和缓存加锁**，然后**执行后面的指令**，在**Lock锁住总线**的时候，其他CPU的读写请求都会**被阻塞**，**直到锁释放**。最后**释放锁后**会把高速缓存中的脏数据全部**刷新回主内存**，且这个**写回内存的操作**会使在其他CPU里**缓存了该地址的数据无效**。

 **lock前缀指令相当于一个内存屏障**内存屏障主要提供3个功能：

1. 确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；
2. **强制将对缓存的修改操作立即写入主存**，利用缓存一致性机制，并且缓存一致性机制会阻止同时修改由两个以上CPU缓存的内存区域数据；
3. 如果是写操作，它会导致其他CPU**中对应的缓存行无效。**

具体CPU底层我也不是特别熟悉，也是网上找到资料也就到这里，如果网友感兴趣可以在去搜索，或者有大佬直接解答。

##### volatile变量不保证原子性

```java
public class VolatileTest {
    public static volatile int race = 0 ;
    public static void increase(){
        race++;
    }
    private static final int THREAD_COUCNT = 20;

    public static void main(String[] args) {
        Thread[] threads = new Thread[THREAD_COUCNT];
        for (int i = 0; i < THREAD_COUCNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    increase();
                }
            });
            threads[i].start();
        }
        while (Thread.activeCount()>2){
            Thread.yield();
        }
        System.out.println(race);
    }
}   
 //idea 里执行activeCount大于2
```

如果运行正常结果应该使200000，但是实际的结果总是小于200000，而且每次输出都不一样。这种事情出现的原因就是i++不是一个原子性的操作（1、获取i；2、i自增；3、回写i）。

A，B两个线程同时自增i；
由于volatile可见性，因此步骤1线程一定拿到的是最新的i，也就是相同的i。但是从第2步开始就有问题了，有可能出现的场景是线程A自增了i并回写，但是线程B此时已经拿到了i，不会再去拿线程A回写的i，因此对原值进行了一次自增并回写；这就导致了线程非安全，也就是说的多线程计数器结果不对。

如果线程A对i进行自增了以后CPU缓存不是应该通知其他缓存，并且重新load i么？

拿的前提是读，问题是，线程A对i进行了自增，线程B已经拿到了i并不存在需要再次读取i的场景，当然是不会重新load i这个值的。

ps：也就是线程B的缓存行内容的确会失效。但是此时线**程B中i的值已经运行在加法指令中，不存在需要再次从缓存行读取i的场景。**

所以对于原子操作的变量可以使用volatile，如果是多个指令执行的的操作需要，使用synchronized关键字去保证原子性，或者使用`java.util.concurrent.atomic`去操作。

```java
//下面代码是线程安全的
public class VolatileTest {
    public static AtomicInteger race = new AtomicInteger(0);
    public static void increase(){
        race.incrementAndGet();
    }
    private static final int THREAD_COUCNT = 20;

    public static void main(String[] args) {
        Thread[] threads = new Thread[THREAD_COUCNT];
        for (int i = 0; i < THREAD_COUCNT; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    increase();
                }
            });
            threads[i].start();
        }
        while (Thread.activeCount()>2){
            Thread.yield();
        }
        System.out.println(race.get());
    }
}
```

#### 并发的3个特性

我们看完上面围绕并发讨论的内容也不难发现，并发主要也就是围绕三个特性在讨论，原子性，可见性和有序性。

##### 原子性

由于Java内存模型直接保证了原子变量的操作包括read、load、assign、use、store、write这6种。基本的数据类型的访问和读写都是原子操作（i++不是，long和double变量也不是不过long和double可以通多`JVM`底层去处理）。如果Java中需要更大的原子性需求就去使用lock和unlock操作来满足这种需求，尽管lock和unlock操作没有直接开放给用户，但是提供了字节码指令`monitorenter`和`monitorexit`来隐式操作，这两个字节码指令反映到Java代码中就是同步块—`synchronized`关键字。`synchronized`操作保证了原子性

##### 可见性

可见性就是当一个线程修改共享变量的值时，其他线程能够立即知道这个修改，Java内存模型，通过变量修改同步主内存，在变量读取前从主内存刷新变量值，这种依赖主内存做传递媒介的方式实现内存的可见性，无论使普通变量还是volatile变量都是共享变量方式实现可见性。volatile和普通变量的区别使volatile变量通过内存屏障可以保证修改的值立即同步回主内存，同时可以使其他内存中的值失效，保证了volatile在多线程下的可见性（同时也保证了有序性）。而普通变量却无法保证。

除了volatile变量保证了可见性，final和synchronized也保证了可见性。synchronized通过对一个变量unlock操作必须把变量值同步回主内存执行store和write操作实现的，而final则是在构造器中被初始化之后this指针没有传递出去也就是没有逃逸，保证了不会被赋值或者其他线程访问到初始化一半的对象。

##### 有序性

Java程序中天然的有序性可以总结：如果在本线程内观察，所有操作都是有序；如果在其他线程中观察所有操作都是无序的。前半句说的使线程内顺序执行代码，后面指的使指令重排序和工作内存和主内存同步延迟的问题。

Java语言提供了volatile和synchronized两个关键字保证有序性，volatile使通过内存屏障，禁止了指令重排序（编译时候和CPU执行时候），而synchronized则由一个变量在同一个时刻只允许一个线程来进行lock操作实现的（但是可以重入，多次获取锁）这个规定使得同一个锁的同步块只能串行的进入保证了有序。

#### 参考

[一文解决内存屏障](https://monkeysayhi.github.io/2017/12/28/%E4%B8%80%E6%96%87%E8%A7%A3%E5%86%B3%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C/)

[内存屏障与volatile](https://blog.csdn.net/u011663071/article/details/78964991)

[volatile 和 内存屏障](https://www.cnblogs.com/yaowen/p/11240540.html)