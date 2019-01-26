---
title: 浅析Java线程池源码
date: 2018-02-26 09:21:08
tags: [java,多线程]
categories: 多线程
---

##### 什么是线程池

线程池顾名思义，就是有很多线程的一个池子，这里面有多少线程，是要根据你要业务需求来确定；它方便你线程的创建和使用，不需要频繁的创建线程资源使得，线程资源充分的得到利用。所以和数据库链接池类似，线程池的作用就是**充分利用资源，提高相应速度，增加系统的吞吐率同时方便管理和监控线程池中线程使用情况，实现对程序的优化**。当添加的到线程池中的任务超过它的容量时，会有一部分任务阻塞等待。当等待任务超过阻塞队列大小，线程池会通过相应的调度策略和拒绝策略，对添加到线程池中的线程进行管理。

##### 线程池解决什么问题

多线程技术主要解决处理器单元内多个线程执行的问题，它可以显著减少处理器单元的闲置时间，增加处理器单元的吞吐能力。    

假设一个服务器完成一项任务所需时间为：T1 创建线程时间，T2 在线程中执行任务的时间，T3 销毁线程时间。

 如果：T1 + T3 远大于 T2，则可以采用线程池，以提高服务器性能。

一个线程池包括以下**四个基本组成部分**：

1、**线程池管理器**（ThreadPool）：用于创建并管理线程池，包括 创建线程池，销毁线程池，添加新任务；

2、**工作线程**（PoolWorker）：线程池中线程，在没有任务时处于等待状态，可以循环的执行任务；

3、**任务接口**（Task）：每个任务必须实现的接口，以供工作线程调度任务的执行，它主要规定了任务的入口，任务执行完后的收尾工作，任务的执行状态等；  

4、**任务队列**（taskQueue）：用于存放没有处理的任务。提供一种缓冲机制。

 **线程池技术正是关注如何缩短或调整T1,T3时间的技术，从而提高服务器程序性能的。它把T1，T3分别安排在服务器程序的启动和结束的时间段或者一些空闲的时间段，这样在服务器程序处理客户请求时，不会有T1，T3的开销了。**

线程池不仅调整T1,T3产生的时间段，而且它还显著减少了创建线程的数目，看一个例子：
假设一个服务器一天要处理50000个请求，并且每个请求需要一个单独的线程完成。在线程池中，线程数一般是固定的，所以产生线程总数不会超过线程池中线程的数目，而如果服务器不利用线程池来处理这些请求则线程总数为50000。一般线程池大小是远小于50000。所以利用线程池的服务器程序不会为了创建50000而在处理请求时浪费时间，从而提高效率。

##### 线程池如何设计的

- **Core and maximum pool sizes** （ThreadPoolExecutor会根据corePoolSize以及maximumPoolSize的边界自动的调整线程池的大小。）

  1、当通过execute(Runnable)提交任务时，而且**正在运行的线程数少于corePoolSize**，即使其他线程处于空闲状态，也会创建一个新的线程执行这个任务；

  2、如果有**大于corePoolSize但是小于maximumPoolSize**数量的线程正在运行，则新提交的任务会放进workQueue进行任务缓存，但是**如果workQueue已满**，则会**直接创建线程执行**，但是如果**创建的线程数大于maximum pool sizes的时候将拒绝任务**。
  3、**当corePoolSize和maximumPoolSize **相等时则会创建固定数量的线程池

  4、**将maximumPoolSize 设置为无边界的**，比如整数的最大值，则意味着线程数和任务数量一致，也就没有等待的任务
  5、corePoolSize、maximumPoolSize可以根据实际需求通过构造器设置，也可以动态的在运行时设置。

- **On-demand construction** （按照需求构造线程）
  1、默认情况下，每一个核心线程只有当有新任务到来时才会初始化创建，并执行
  2、但是可以在运行时可以通过prestartCoreThread(一个coreThread)或者prestartAllCoreThreads(全部coreThread)来提前创建并运行指定的核心线程，这种需求适用于初始化线程池时，任务队列初始不为空的情况下。

- **Creating new threads** （创建一个新的线程）

  1、创建线程是通过**ThreadFactory**。除非特别的设定，否则默认使用Executors.defaultThreadFactory作为线程池，这个线程池创建的所有线程都有相同的线程组，线程优先级，非守护线程的标志
  2、通过应用不同的线程池，可以更改线程的名字，线程组，优先级，守护标志等等
  3、当通过**newThread()**调用线程池创建线程池失败时，返回null，此时执行器会继续运行，但是可能处理不了任何任务

  4、线程需要处理"modifyThread" RuntimePermission，对线程修改进行运行时权限检查。如果使用这个线程池的工作线程或者其他线程没有处理这个认证"permission"则会使服务降级：对于线程池的所有设置都不会及时的生效，一个已经关闭的线程池可能还会处于一种线程池终止没有完成的状态

- **Keep-alive times** (空闲的线程存活时间)

  1、当这个线程池此时含有多余corePoolSize的线程存在，则多余的线程在空闲了超过keepAliveTime的时间将会被终止
  2、这提供了一种减少空闲线程从而降低系统线程资源损耗的方法，还可以通过setKeepAliveTime进行动态设置
  3、**默认情况下，keep-alive policy只对超出corePoolSize的线程起作用**，但是可以通过方法**allowCoreThreadTimeOut(boolean)**将空闲超时策略同样应用于coreThread，但是要保证超时时间不为0值。

- **Queue** （阻塞队列，任何BlockingQueue都可以被用来容纳和传递提交的任务）

  1、如果正在运行的线程小于corePoolSize，则executor会新增一个线程而不是将任务入队
  2、如果正在运行的线程大于corePoolSize但是小于maximumPoolSize，executor则会将任务入队，而不是创建一个线程
  3、如果任务不能入队（队列已满），则在没有超出maximumPoolSize的情况下创建一个新的线程，否则某种拒绝策略拒绝这个任务。

- **three general strategies for queuing**  （三种入队策略）
  1、**Direct handoffs：直接传递**。比如 **synchronousQueue**，这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。
  2、**Unbounded queues：无界队列。**比如 没有指定容量的**LinkedBlockingQueue**，这将会使coreThread一直工作，而且由于任务总能入队，所以也不会创建其他超过corePoolSize的线程。用于所有任务完全独立，不相关，比如平滑瞬间高并发web页面的请求等，其实相当于异步框架了
  3、**Bounded queues：有界队列。** 比如**ArrayBlockingQueue**，有助于在设置有限大的maximumPoolSizes时，阻止造成系统资源的枯竭。队列大小和最大池大小可能需要相互折衷：使用**大队列和小池最大限度地减少CPU的使用，操作系统资源，和上下文切换开销，但可能会导致人为的低吞吐量**。**如果任务经常被阻塞（例如，如果它们是I/O绑定），系统可能比你允许的时间安排更多线程的时间**。使用**小队列通常需要更大的池大小，这使得CPU繁忙，但可能会遇到不可接受的调度开销，这也降低吞吐量  **

- **Rejected tasks** （拒绝任务）

  当提交一个新任务时，如果Executor已经关闭或者有限的workQueue，maximumPoolSizes，并且他们已经饱和了，只要出现其中一种情况都会被拒绝。有四种已经定义的处理策略。也可以继承RejectedExecutionHandler自定义实现

- **Hook methods** （钩子方法，提供在每个任务执行时不同阶段执行不同的处理函数）

  1、**protected void beforeExecute(Thread t, Runnable r)：**优先使用指定的线程处理给定的任务，并在任务执行前做一些处理（如设置ThreadLocal变量或者记录一些日志等），t为执行r任务的线程，r为提交的任务。

  2、**protected void afterExecute(Runnable r, Throwable t)：**任务执行完成时处理。r为执行完的任务，t为指定的造成任务终止的异常，如果设置为null则执行会正常完成，不会抛出异常

  3、**protected void terminated()**当Executor终止时，被调用一次

  **以上三个方法都为空方法**，使用者自行实现。在进行多层嵌套时都要显示调用 **super.method()** 完成上层的处理函数。如果在调用方法时发生异常，则内部的工作线程可能会依次失败，突然终止。

  可以继承ThreadPoolExecute，并实现上述几个Hook方法来检测线程池的状态，自定义自己的线程池，如监控任务的平均、最大、最小执行时间，来发现有没有一致阻塞的线程任务。

##### 线程池是如何实现的

在Java中线程池使用ThreadPoolExecutor这类去实现了，它里面封装了线程池的相关属性和创建线程池的基本方法。下面我们就来简单看一下源代码，简答的分析一波

```java
    /* 这些是Java线程池中阶一些基本变量和简单方法*/
	//用来存储工作线程数和工作状态，初始化状态和数量，状态为RUNNING，线程数为0
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	//用来计数工作线程数。workerCount的最大为2^29 -1。这里Java中Integer是32位
    private static final int COUNT_BITS = Integer.SIZE - 3;
	//这里取得后29位，也就是线程池的容量
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits(线程池的运行状态是存储在高3位)
    //可以接受新的任务，也可以处理阻塞队列里的任务
    private static final int RUNNING    = -1 << COUNT_BITS;
	//b不可以接受新的任务，可以处理阻塞队列里的任务
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
	//不接受新的任务，不处理阻塞队列里的任务，中断正在处理的任务
    private static final int STOP       =  1 << COUNT_BITS;
	//过渡状态，也就是说所有的任务都执行完了，当前线程池已经没有有效的线程，这个时候线程池的状态将会TIDYING，并且将要调用terminated方法
    private static final int TIDYING    =  2 << COUNT_BITS;
	//终止状态。terminated方法调用完成以后的状态
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl (打包和拆包ctl变量，也就是获取存储工作线程数和工作状态的变量和把工作状态和线程数转化成ctl)这里都是通过位运算实现的。
	//获取线程池的运行状态,根据ctl。CAPACITY的非操作得到的二进制位11100000000000000000000000000000，然后做在一个与操作，相当于直接取前3位的的值
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
	//获取工作的线程数，根据ctl。也就是后29位的数字。 直接跟CAPACITY做一个与操作即可，CAPACITY就是的值就 1 << 29 - 1 = 00011111111111111111111111111111。 与操作的话前面3位肯定为0，相当于直接取后29位的值
    private static int workerCountOf(int c)  { return c & CAPACITY; }
	//根据运行状态和工作线程数获取ctl。或操作
    private static int ctlOf(int rs, int wc) { return rs | wc; }
	
	//不需要拆包的去访问变量，这些状态依赖runState
    /*
     * Bit field accessors that don't require unpacking ctl.
     * These depend on the bit layout and on workerCount being never negative.
     */
	//来判断两个运行状态的大小，运行状态越大，越接近停止。
    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }
	//来判断两个运行状态的大小
    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }
	//来判当前状态是否是运行状态
    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }
```

线程池中使用**AtomicInteger **的CAS机制来实现对**运行时状态以及工作线程计数的并发一致性操作**，低29位（32-3）用来保存workerCount，所以workerCount的最大为2^29 -1 。高3位用来保存runState，这样实现具有较高效率，不用单独两次存储。

**RUNNING -> SHUTDOWN**：手动调用shutdown方法，或者ThreadPoolExecutor要被GC回收的时候调用finalize方法，finalize方法内部也会调用shutdown方法

**(RUNNING or SHUTDOWN) -> STOP**：调用shutdownNow方法

**SHUTDOWN -> TIDYING**：当队列和线程池都为空的时候

**STOP -> TIDYING**：当线程池为空的时候

**TIDYING -> TERMINATED**：terminated方法调用完成之后，ThreadPoolExecutor内部还保存着线程池的有效线程个数。

```java
   /**
     * Attempts to CAS-increment the workerCount field of ctl.
     */
	//尝试CAS-递增ctl的workerCount字段。
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }

    /**
     * Attempts to CAS-decrement the workerCount field of ctl.
     */
	//尝试CAS-递减ctl的workerCount字段。
    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }

    /**
     * Decrements the workerCount field of ctl. This is called only on
     * abrupt termination of a thread (see processWorkerExit). Other
     * decrements are performed within getTask.
     */
	//减少ctl的workerCount字段的值，当一个线程因为异常退出的时候调用这个方法。其他的递减都在getTask方法里进行
    private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }

	//缓存队列，是一个存放没有处理的任务。（也就是runnable接口中写的代码）
    private final BlockingQueue<Runnable> workQueue;

  	// 线程池主锁，用于访问worker线程集，还有其他关于线程池信息的记录信息（比如线程池大小，runState）
    private final ReentrantLock mainLock = new ReentrantLock();

    //工作线程集合，访问时需获取mainLock 
    private final HashSet<Worker> workers = new HashSet<Worker>();

   // mainLock上的终止条件量，用于支持awaitTermination
    private final Condition termination = mainLock.newCondition();
 	//记录线程最大的线程数，访问时需获取mainLock 
    private int largestPoolSize;
   //记录已经完成的记录数，访问时需获取mainLock 
    private long completedTaskCount;

    /**
     * 以下所有变量都为volatile类型的，以便能使所有操作都基于最新值 
     * （因为这些值都可以通过对应的set方法，在运行时动态设置），
     * 但是不需要获取锁，因为没有内部不变量依赖于它们与其他动作同步变化。
     */ 

   // 用于创建新线程的线程工厂
    private volatile ThreadFactory threadFactory;

   //当线程池满了的时候，拒绝服务策略
    private volatile RejectedExecutionHandler handler;
   //一个空闲线程，可以保持存活的最大时间。核心线程不算
    private volatile long keepAliveTime;

	//默认是false，核心线程可以在空闲的时候存活，如果是true那么核心线程超时会关闭。
    private volatile boolean allowCoreThreadTimeOut;

	//核心线程个数，如果allowCoreThreadTimeOut为false那么会一直存在，如果ture的话可能为0。最小
    private volatile int corePoolSize;

	//线程池的最大线程数。它小于CAPACITY
    private volatile int maximumPoolSize;

    //设置默认的拒绝策略。
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();

	// 对于调用线程池的shutdown(),shutdownNow()方法权限认证。用于保护线程安全的
    private static final RuntimePermission shutdownPerm =
        new RuntimePermission("modifyThread");
```

线程池的几个构造函数，**线程池中一共有7个参数。每个参数都代表这个不同的意义。corePoolSize核心线程的数量；maximumPoolSize线程池的最多线程数；keepAliveTime当一个线程在空闲时的存活时间（核心线程要看allowCoreThreadTimeOut参数）；TimeUnit时间单位多数是秒，也可以设置毫秒；workQueue缓冲队列（即线程池没有运行的任务队列）；threadFactory用于创建新线程的线程工厂；handler用于线程池在满负荷下的拒绝策略。可以使抛异常，也可是放弃任务等。** 拒绝策略指的是由于超出线程范围和队列容量而使执行被阻塞时所使用的处理程序。

```java
	//使用默认的线程工厂和，默认的拒绝策略生产线程池
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
      	//调用构造函数
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
	//使用默认的拒绝策略，传入自定义线程工厂
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
      //调用构造函数
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }
	//使用默认的线程工厂，使用其他拒绝策略
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }

   //线程池的构造函数
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
      	//如果核心线程数小于0，最大线程数小于等于0，最大线程数小于核心线程数，超时时间小于0会抛出非法参数异常
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
      	//当工作队列为null，线程工厂为null，拒绝策略为null抛出空指针异常。
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

线程池的几个内部类，主要是Worker类，和几个拒绝策略的实现类。Worker是一个AQS的实现类(为何设计成一个AQS在闲置Worker里会说明)，同时也是一个实现Runnable的类，实现独占锁(非重入的互斥锁)，它的构造函数只接受一个Runnable参数，内部保存着这个Runnable属性，还有一个thread线程属性用于包装这个Runnable(这个thread属性使用ThreadFactory构造。在构造函数内完成thread线程的构造)，实现互斥锁主要目的是为了中断的时候判断线程是在空闲还是运行（**判断是否是闲置线程，是否可以被强制中断。** 一般有锁闲置的工作线程，因为在执行runWorker的时候会去掉锁），可以看后面 shutdown 和 shutdownNow 方法的分析。另外还有一个completedTasks计数器表示这个Worker完成的任务数。Worker类复写了run方法，使用ThreadPoolExecutor的runWorker方法(在addWorker方法里调用)，直接启动Worker的话，会调用ThreadPoolExecutor的runWork方法。**需要特别注意的是这个Worker是实现了Runnable接口的，thread线程属性使用ThreadFactory构造Thread的时候，构造的Thread中使用的Runnable其实就是Worker。**在前面还有一个HashSet的Worker 的集合workers，线程池通过管理线程池里的线程。

```java
·	//worker是用于管理
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        //这个类永远都不会序列化，只是符合Javac的规范
        private static final long serialVersionUID = 6138294804551838833L;

       //当前的这个worker运行在这个Thread，worker也实现Runnable接口。如果ThreadFactory创建失败可能是null
        final Thread thread;
       //初始化运行任务，可能是空
        Runnable firstTask;
        //每线程任务计数器
        volatile long completedTasks;

       //Worker的构造函数，通过Worker的firstTask指定一个任务，同时通过ThreadFactory创建一个线程
        Worker(Runnable firstTask) {
          	//把状态位设置成-1，这样任何线程都不能得到Worker的锁，除非调用了unlock方法。这个unlock方法会在runWorker方法中一开始就调用，这是为了确保Worker构造出来之后，没有任何线程能够得到它的锁，除非调用了runWorker之后，其他线程才能获得Worker的锁
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
          // 使用ThreadFactory构造Thread，这个构造的Thread内部的Runnable就是本身，也就是这个Worker。所以得到Worker的thread并start的时候，会执行Worker的run方法，也就是执行ThreadPoolExecutor的runWorker方法
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker 将主运行循环委托给外部runWorker */
        public void run() {
            runWorker(this);
        }

        // Lock methods 锁方法,state属性是从AQS中获取
        //
        // The value 0 represents the unlocked state.
      	//值为0 的时候是非锁定状态
        // The value 1 represents the locked state.
		//值为1的时候是锁定状态
      	//是否被锁定，如果有就不等于0。
        protected boolean isHeldExclusively() {
            return getState() != 0;
        }
	   //尝试获取锁
        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
	   //释放锁
        protected boolean tryRelease(int unused) {
          	//设定当前资源的线程为null
            setExclusiveOwnerThread(null);
          	//设置状态为0
            setState(0);
            return true;
        }
		//锁定
        public void lock()        { acquire(1); }
      	//尝试获取锁
        public boolean tryLock()  { return tryAcquire(1); }
      	//释放锁，通过内部函数实现，然后调用
        public void unlock()      { release(1); }
      	//判断是否被锁定
        public boolean isLocked() { return isHeldExclusively(); }
		
      	//中断已经开始的线程
        void interruptIfStarted() {
            Thread t;
          	//判断任务已经开始，然后中断执行的线程（这个中断是强制的）
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }

 	//直接运行新添加的任务，除非意外终止。
    public static class CallerRunsPolicy implements RejectedExecutionHandler {
         //构造函数
        public CallerRunsPolicy() { }

     	//直接运行任务
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }

    //超出线程范围和队列容量时抛出异常
    public static class AbortPolicy implements RejectedExecutionHandler {
       
       //构造函数
        public AbortPolicy() { }
		//抛出异常
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }

	//拒绝任务的处理程序，丢弃被拒绝的任务
    public static class DiscardPolicy implements RejectedExecutionHandler {
       	//构造函数
        public DiscardPolicy() { }

       //什么都不做，这会有丢弃任务r的效果
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }

	//丢弃最老的任务
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        //构造函数
        public DiscardOldestPolicy() { }

       
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
          //如果线程池在运行，那么就获取缓冲队列，抛出最上面的一个  
          if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```

ThreadPoolExecutor执行任务。首先通过submit或者excute方法把任务放到线程池中（这里如果线程池空闲会直接执行，否则会进入到缓冲队列中去），然后线程池从缓冲队列中获取任务（getTask），然后添加Work，最后执行要执行任务内容(runWorker)。

```java
// submit是存在AbstractExecutorService的源码。一般在Executors中会用到
public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
  		//newTaskFor是把runnable接口转换有返回值
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }

//很明显地看到，submit方法内部使用了execute方法，而且submit方法是有返回值的。在调用execute方法之前，使用FutureTask包装一个Runnable，这个ftask就是返回值。
```

由于submit方法内部调用execute方法，所以execute方法就是执行任务的方法，来看一下execute方法，execute方法内部分3个步骤进行处理。

1. 如果当前正在执行的Worker数量比corePoolSize(基本大小，核心线程数)要小。直接创建一个新的Worker执行任务，会调用addWorker方法
2. 如果当前正在执行的Worker数量大于等于corePoolSize(基本大小，核心线程数)。将任务放到阻塞队列里，如果阻塞队列没满并且状态是RUNNING的话，直接丢到阻塞队列，否则执行第3步。丢到阻塞队列之后，还需要再做一次验证(丢到阻塞队列之后可能另外一个线程关闭了线程池或者刚刚加入到队列的线程死了)。如果这个时候线程池不在RUNNING状态，把刚刚丢入队列的任务remove掉，调用reject方法，否则查看Worker数量，如果Worker数量为0，起一个新的Worker去阻塞队列里拿任务执行
3. 丢到阻塞失败的话，会调用addWorker方法尝试起一个新的Worker去阻塞队列拿任务并执行任务，如果这个新的Worker创建失败，调用reject方法

上面说的Worker可以暂时理解为一个执行任务的线程。

```java
 public void execute(Runnable command) {
   		//判断任务是否为空
        if (command == null)
            throw new NullPointerException();
        //获取ctl
        int c = ctl.get();
   		//根据ctl获取工作线程线程数，如果小于核心线程数，添加新的worker，也就是新线程
        if (workerCountOf(c) < corePoolSize) { // 第一个步骤，满足线程池中的线程大小比基本大小要小
            if (addWorker(command, true)) // addWorker方法第二个参数true表示使用基本大小
                return;
          	//否则重新获取ctl
            c = ctl.get();
        }
   		
        if (isRunning(c) && workQueue.offer(command)) { // 第二个步骤，线程池的线程大小比基本大小要大，并且线程池还在RUNNING状态，阻塞队列也没满的情况，加到阻塞队列里
          	//重新获取ctl，二次校验。
            int recheck = ctl.get();
           // 虽然满足了第二个步骤，但是这个时候可能突然线程池关闭了，所以再做一层判断
            if (! isRunning(recheck) && remove(command))//关闭了
                reject(command); //调用拒绝策略方法
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false); // 第三个步骤，直接使用线程池最大大小。addWorker方法第二个参数false表示使用最大大小
        }
        else if (!addWorker(command, false))
          	//调用拒绝策略方法
            reject(command);
    }
```

如何添加一个Worker。

1. **在外循环对运行状态进行判断**，**内循环通过CAS机制对workerCount进行增加**，当设置成功，则跳出外循环，否则进行进行内循环重试
2. **外循环之后，获取全局锁，再次对运行状态进行判断，符合条件则添加新的工作线程，并启动工作线程**，如果在最后对添加线程没有开始运行（可能发生**内存溢出**，操作系统**无法分配线程**等等）则对**添加操作进行回滚**，移除之前添加的线程

```java
 
	// 两个参数，firstTask表示需要跑的任务。boolean类型的core参数为true的话表示使用线程池的基本大小，为false使用线程池最大大小
    //返回值是boolean类型，true表示新任务被接收了，并且执行了。否则是false  
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
          	//获取ctl
            int c = ctl.get();
          	//获取线程池状态
            int rs = runStateOf(c);

        // 这个判断转换成 rs >= SHUTDOWN && (rs != SHUTDOWN || firstTask != null || workQueue.isEmpty)。 
        // 概括为3个条件：
        // 1. 线程池不在RUNNING状态并且状态是STOP、TIDYING或TERMINATED中的任意一种状态

        // 2. 线程池不在RUNNING状态，线程池接受了新的任务 

        // 3. 线程池不在RUNNING状态，阻塞队列为空。  满足这3个条件中的任意一个的话，拒绝添加
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
		
            for (;;) {
              	//获取工作线程数
                int wc = workerCountOf(c);
              	// 1、工作线程数大于总容量；2、如果core是true大于核心线程数，如果是false大于设置最大线程数，如果满足就返回false，不能添加
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
              // 没有超过各种大小的话，cas操作线程池线程数量+1，cas成功的话跳出循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // 重新检查状态
                if (runStateOf(c) != rs) // 如果状态改变了，重新循环操作
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
      
	   // 走到这一步说明cas操作成功了，线程池线程数量+1
        boolean workerStarted = false;// 任务是否成功启动标识
        boolean workerAdded = false;// 任务是否成功添加标识
        Worker w = null;
        try {
            w = new Worker(firstTask); //基于任务firstTask构造worker，firstTask可能是null
            final Thread t = w.thread; //从worker中获取线程对象
            if (t != null) { //ThreadFactory构造出的Thread有可能是null，做个判断
                final ReentrantLock mainLock = this.mainLock;// 得到线程池的可重入锁
              	// 锁定下面的操作
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                  	//获取线程池运行状态
                    int rs = runStateOf(ctl.get());
				  //1、线程池的状态是RUNNING的；
                   //2、线程池的状态是SHUTDOWN同时firstTask是空
                   //上面两条有一个满足都可以天剑，否则添加失败
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // 判断线程是否还活着，也就是说线程已经启动并且还没死掉
                            throw new IllegalThreadStateException();
                        workers.add(w); // worker添加到线程池的workers属性中，是个HashSet
                      	//获取workers的大小
                        int s = workers.size();
                        if (s > largestPoolSize)//如果workers的大小小于largestPoolSize，s付给线程最大值
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                  	//释放锁
                    mainLock.unlock();
                }
              	//如果worker添加成功
                if (workerAdded) {
                  	//开始执行线程程序
                    t.start();
                  	//设置标志为true
                    workerStarted = true;
                }
            }
        } finally {
          	//如果启动失败
            if (! workerStarted)
              	//调用添加失败方法
                addWorkerFailed(w);
        }
      	//返回运行（也就是添加worker）是否成功失败
        return workerStarted;
    }

    	
	//添加工作线程失败后调用的方法
    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock; //获取可重入锁
        mainLock.lock(); //锁定代码
        try {
            if (w != null) //如果添加的工作线程不为空，从workers中移除该工作线程
                workers.remove(w);
          	// 减少工作线程计数
            decrementWorkerCount();
            // 因为中断异常而没有启动线程，从而回滚已入队的线程
            // 这个中断异常可能是关闭线程池时发生的，所以应该将终止线程池的信号传播
            tryTerminate();//终止线程池
        } finally {
            mainLock.unlock();
        }
    }
```

运行Worker中的任务。

线程池中的这个基本大小指的是Worker的数量。一个Worker是一个Runnable的实现类，会被当做一个线程进行启动。Worker内部带有一个Runnable属性firstTask，这个firstTask可以为null，为null的话Worker会去阻塞队列拿任务执行，否则会先执行这个任务，执行完毕之后再去阻塞队列继续拿任务执行。

所以说如果Worker数量超过了基本大小，那么任务都会在阻塞队列里，当Worker执行完了它的第一个任务之后，就会去阻塞队列里拿其他任务继续执行。

Worker在执行的时候会根据一些参数进行调节，比如Worker数量超过了线程池基本大小或者超时时间到了等因素，这个时候Worker会被线程池回收，线程池会尽量保持内部的Worker数量不超过基本大小

Worker执行任务的时候调用的是Runnable的run方法，而不是start方法，调用了start方法就相当于另外再起一个线程了。

```java
//这是个final的方法，不能被重写。 
final void runWorker(Worker w) {
  		// 获取当前线程，在Worker类中的run方法调用这个runWorker方法。
        Thread wt = Thread.currentThread();
  	    // 从worker类中获取任务
        Runnable task = w.firstTask;
        w.firstTask = null;//清空worker中的任务
        w.unlock(); // 释放worker上独占锁。
        boolean completedAbruptly = true;// 因为运行异常导致线程突然终止的标志
        try {
          // 如果worker中的任务不为空，继续，否则使用getTask获得任务。一直死循环，除非得到的任务为空才退出
            while (task != null || (task = getTask()) != null) {
              	// 如果拿到了任务，给自己上锁，表示当前Worker已经要开始执行任务了，已经不是闲置Worker(闲置Worker的解释请看下面的线程池关闭)
                w.lock();
               // 在执行任务之前先做一些处理。 
              //1. 如果线程池已经处于STOP状态并且当前线程没有被中断，中断线程 
              //2. 如果线程池还处于RUNNING或SHUTDOWN状态，并且当前线程已经被中断了，重新检查一下线程池状态，如果处于STOP状态并且没有被中断，那么中断线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                  	//执行任务前的函数。一般都是是自己实现，它提供了空方法，需要自己复写
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                      // 运行任务.这里调用的不是start方法，而是run方法。
                      //这里run的时候可能会被中断，比如线程池调用了shutdownNow方法
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                      //执行运行前的函数。一般都是是自己实现，它提供了空方法，需要自己复写
                       afterExecute(task, thrown);
                    }
                } finally {
                    task = null; //清空任务
                    w.completedTasks++; //已完成任务数+1
                    w.unlock(); //释放worker的锁
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly); // 回收Worker方法
        }
    }
    
   //私有方法，用于回收Worker              
   private void processWorkerExit(Worker w, boolean completedAbruptly) {
     	//如果突然情况导致线程终止，WorkerCount-1
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();
		//获取锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
          	//完成的任务数增加
            completedTaskCount += w.completedTasks;
          	//移除完成的workers
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }
		//尝试终止线程池
        tryTerminate();
	    //获取ctl
        int c = ctl.get();
     	//根据ctl获取状态，判断是否是停止状态之前的状态Stop 为1
        if (runStateLessThan(c, STOP)) {
          	//如果线程没有异常退出
            if (!completedAbruptly) {
              	//allowCoreThreadTimeOut为true 的时候是，工作线程最小为0，否则最小为核心工作线程个数
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
              // 如果线程数量大于等于正常工作的数量则不再添加新的线程
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
          	//添加一个工作线程（没有任务的），false是使用最大线程数。true是核心线程数
            addWorker(null, false);
          // 新开一个Worker代替原先的Worker
          // 新开一个Worker需要满足以下3个条件中的任意一个：
          // 1. 用户执行的任务发生了异常
          // 2. Worker数量比线程池基本大小要小
          // 3. 阻塞队列不空但是没有任何Worker在工作
        }
    }
         
                 
```

获取任务getTask，一般会在runWorker的时候去调用。

1. 通过**死循环来对线程池状态进行判断，并获取任务，在超时发生之前发生中断则重置超时标志位false并进行重试**，如果获取到任务则返回任务
2. 主要来看一下是**如何实现移除空闲keepAliveTime线程的**：`workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)`方法从任务队列中**定时获取任务**，**如果超时，则说明线程已经在等待了keepAliveTime都没有获得任务**，则将**超时标志设为true**，在下一次循环时进行判断，如果发现上一次获取任务发生超时，则立刻返回null，这时worker线程主循环将正常结束，并移除结束的worker。

```java
// 获取任务
private Runnable getTask() {
  		
        boolean timedOut = false; // 如果使用超时时间并且也没有拿到任务的标识

        for (;;) {
            int c = ctl.get(); //获取ctl
            int rs = runStateOf(c); //根据ctl获取线程状态，每次现用现取
			// 线程池状态在SHUTDOWN，或者是Stop之后或者工作队列为空直接返回null
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();//工作线程数-1
                return null;
            }
		   
            int wc = workerCountOf(c); //获取工作线程数

            // Are workers subject to culling?
            //timed只的是，是否有直接的线程可用，是否要等待。
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		   //工作线程个数大于最大线程池数，或者等待超时，后者阻塞队列中为空
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c)) //worker数量减一，返回null。回收worker
                    return null;
                continue; //否则继续获取
            }

            try {
              	//timed为true说明线程池里没有充足的线程需要等待，否则直接获取任务
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();// 如果在keepAliveTime时间内获取到任务则返回,
                if (r != null)
                    return r;
              // 否则将超时标志设置为true
                timedOut = true;
            } catch (InterruptedException retry) {
              	//抛异常设置超时时间为false表示没有超时
                timedOut = false;
            }
        }
    }
```

Worker在回收的时候会尝试终止线程池也就是tryTerminate方法。尝试关闭线程池的时候，会检查是否还有Worker在工作，检查线程池的状态，没问题的话会将状态过度到TIDYING状态，之后调用terminated方法，terminated方法调用完成之后将线程池状态更新到TERMINATED。

```java
 //线程池尝试结束自己运行，不一定会结束
 // 满足3个条件中的任意一个，不终止线程池
 // 1. 线程池还在运行，不能终止
 // 2. 线程池处于TIDYING或TERMINATED状态，说明已经在关闭了，不允许继续处理
 // 3. 线程池处于SHUTDOWN状态并且阻塞队列不为空，这时候还需要处理阻塞队列的任务，不能终止线程池
   
final void tryTerminate() {
        for (;;) {
          	//获取ctl
            int c = ctl.get();
          	//上面的三种情况，直接返回
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
          	//走到这一步说明线程池已经不在运行，阻塞队列已经没有任务，但是还要回收正在工作的Worker
            if (workerCountOf(c) != 0) { // Eligible to terminate
              // 由于线程池不运行了，调用了线程池的关闭方法.
              // 中断闲置Worker，直到回收全部的Worker。这里没有那么暴力，只中断一个，中断之后退出方法，中断了Worker之后，Worker会回收，然后还是会调用tryTerminate方法，如果还有闲置线程，那么继续中断法
                interruptIdleWorkers(ONLY_ONE);
                return;
            }
		   // 获取锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
              	//设置线程池状态，和工作线程个数
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                      	//调用terminated方法，结束线程池
                        terminated();
                    } finally {
                      	//设置线程池状态和工作线程个数
                        ctl.set(ctlOf(TERMINATED, 0));
                      	//唤醒其他线程
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```

线程池的关闭主要是两个方法，shutdown和shutdownNow方法。

shutdown方法会更新状态到SHUTDOWN，不会影响阻塞队列里任务的执行，但是不会执行新进来的任务。同时也会回收闲置的Worker，闲置Worker的定义上面已经说过了。

shutdownNow方法会更新状态到STOP，会影响阻塞队列的任务执行，也不会执行新进来的任务。同时会回收所有的Worker。

线程池的结束相关的代码

```java
// 将线程池从运行状态转为SHUTDOWN状态
public void shutdown() {
  		//获取锁锁定这块代码
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();// 检查关闭线程池的权限
            advanceRunState(SHUTDOWN);//设置运行状态为SHUTDOWN
            interruptIdleWorkers();//中断空闲的工作线程
		   //钩子方法，默认不处理。ScheduledThreadPoolExecutor会做一些处理
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();//尝试关闭线程池
    }
//将线程池从运行状态转为STOP状态，同时返回阻塞对列中的任务集合
public List<Runnable> shutdownNow() {
  		//要返回的阻塞队列中的任务
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;//获取锁并锁定
        mainLock.lock();
        try {
            checkShutdownAccess();//检验权限
            advanceRunState(STOP);//设置状态为STOP
            interruptWorkers();//关闭运行的工作线程，无论是否闲置
            tasks = drainQueue(); //将队列中没有运行的任务取出来
        } finally {
            mainLock.unlock();
        }
        tryTerminate();//尝试关闭线程池
        return tasks;
    }
 //awaitTermination方法将状态设置为TERMINATED，并拒绝新增任务，调用isShutdown返回true，但是调用isTerminaed返回false，超时等待所有提交的任务的完成。
 public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
   		//将设置时间转化成纳秒
        long nanos = unit.toNanos(timeout);
   		//获取锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (;;) {，
              	//判断线程池状态，如果所有提交的任务已经完成，则立刻返回true
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                if (nanos <= 0)// 已经超时，则返回false
                    return false;
                // 进入等待，直到被通知、中断、超时，则返回剩余的间，
                // 如果返回值小于等于0，则表示是超时返回
                nanos = termination.awaitNanos(nanos);
            }
        } finally {
            mainLock.unlock();
        }
    }
//中断空闲的Worker，传入了参数false，表示要中断所有的正在运行的闲置Worker，如果为true表示只打断一个闲置Worker
private void interruptIdleWorkers(boolean onlyOne) {
  		//获取锁。
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
          	//从workers中取出worker并中断
            for (Worker w : workers) {
              	//获取线程
                Thread t = w.thread;
              	//如果该运行的工作线程是没有打断，同时闲置的工作线程（存在锁的）
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                      	//打断该工作线程
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();//释放worker的锁
                    }
                }
              	//只打断一个，然后跳出循环
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
  //中断workers
  private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
    	//获取锁
        mainLock.lock();
        try {
          	//for循环关闭workers
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
//  在gc之前调用线程池的shutdown()方法,释放相关资源
 protected void finalize() {
        shutdown();
    }
```

interruptIdleWorkers方法，注意，这个方法打断的是闲置Worker，打断闲置Worker之后，getTask方法会返回null，然后Worker会被回收。怎么判断Worker是闲置呢？

闲置Worker是这样解释的：Worker运行的时候会去阻塞队列拿数据(getTask方法)，拿的时候如果没有设置超时时间，那么会一直阻塞等待阻塞队列进数据，这样的Worker就被称为闲置Worker。由于Worker也是一个AQS，在runWorker方法里会有一对lock和unlock操作，这对lock操作是为了确保Worker不是一个闲置Worker。所**以Worker被设计成一个AQS是为了根据Worker的锁来判断是否是闲置线程，是否可以被强制中断**（而且这个锁还是一个不可重入锁，即独占锁）。



##### Java提供了那几种线程池

在Java 的Executors类中有很多创建好的线程池。FixedThreadPool固定大小的线程池，SingleThreadExecutor只有一个线程的线程池，CachedThreadPool带有缓存功能的线程池，ScheduledThreadPool可以定时的线程池，SingleThreadScheduledExecutor只有一个线程并且可以定时的线程池，WorkStealingPool使用ForkJoin方式的线程池,unconfigurableExecutorService不可配置的线程池。

- FixedThreadPool线程池

```java
//FixedThreadPool是一个指定固定大小的线程池，它在构造函数中指定了线程池的大小
public static ExecutorService newFixedThreadPool(int nThreads) {
  		//这调用的是ThreadPoolExecutor的构造函数，指定基本线程和最大线程都是nThreads，空闲线程存活时间是0ms，一般如果不设置allowCoreThreadTimeOut，keepAliveTime不会对基本线程起作用。时间单位是ms。传入一个没有边界的阻塞队列
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
//可以自己指定ThreadFactory
  public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```

- SingleThreadExecutor

```java
// SingleThreadExecutor 创建一个线程的线程池。当单一线程抛出异常停止时会在新建一个线程
public static ExecutorService newSingleThreadExecutor() {
  		//包装一下线程池使他只能暴露ExecutorService的方法，而不能设置线程池的其他参数。这样的目的是为了让任务串行化执行，不会被设置线程池大小
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }

	//只暴露ExecutorService方法的包装类 ExecutorService实现，这里使用了代理方法。
    static class DelegatedExecutorService extends AbstractExecutorService {
        private final ExecutorService e;
        DelegatedExecutorService(ExecutorService executor) { e = executor; }
        public void execute(Runnable command) { e.execute(command); }
        public void shutdown() { e.shutdown(); }
        public List<Runnable> shutdownNow() { return e.shutdownNow(); }
        public boolean isShutdown() { return e.isShutdown(); }
        public boolean isTerminated() { return e.isTerminated(); }
        public boolean awaitTermination(long timeout, TimeUnit unit)
            throws InterruptedException {
            return e.awaitTermination(timeout, unit);
        }
        public Future<?> submit(Runnable task) {
            return e.submit(task);
        }
        public <T> Future<T> submit(Callable<T> task) {
            return e.submit(task);
        }
        public <T> Future<T> submit(Runnable task, T result) {
            return e.submit(task, result);
        }
        public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
            throws InterruptedException {
            return e.invokeAll(tasks);
        }
        public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                             long timeout, TimeUnit unit)
            throws InterruptedException {
            return e.invokeAll(tasks, timeout, unit);
        }
        public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
            throws InterruptedException, ExecutionException {
            return e.invokeAny(tasks);
        }
        public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                               long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException, TimeoutException {
            return e.invokeAny(tasks, timeout, unit);
        }
    }
	
    static class FinalizableDelegatedExecutorService
        extends DelegatedExecutorService {
        FinalizableDelegatedExecutorService(ExecutorService executor) {
            super(executor);
        }
        protected void finalize() {
            super.shutdown();
        }
    }

```

- CachedThreadPool

```java
//创建一个带有缓存队列的线程池
public static ExecutorService newCachedThreadPool() {
  		//基本线程数（corePoolSize）为0，最大线程数是Integer的最大值，keepAliveTime时间是60，单位是秒，它使用的是直接队列，会直接分配给工作线程，如果有空闲工作线程就直接复用，没有就会直接新建一个线程，不会阻塞。一个工作线程空闲60秒后会被回收
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

- ScheduledThreadPool

```java
//创建一个可以定时的线程池,并指定线程池的大小
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
//ScheduledThreadPoolExecutor 是继承ThreadPoolExecutor所以super方法就是ThreadPoolExecutor构造函数
 public ScheduledThreadPoolExecutor(int corePoolSize) {
   		//可以指定线程池的基本大小。线程池最大值是integer 的最大值，他的keepAliveTime是0，时间单位是纳秒。队列是一个延时队列。可也设置时间参数。
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```

- SingleThreadScheduledExecutor

```java
//创建单一线程的线程池，并且可以定时执行任务。当单一线程抛出异常停止时会在新建一个线程
public static ScheduledExecutorService newSingleThreadScheduledExecutor() {
  		//这里创建的基本大小为1的定时线程池
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1));
    }
```

- WorkStealingPool

```java
// 创建一个ForkJoin的线程池，也就是将好多任务分解多个任务后给多个线程执行，执行之后在汇总。Fork获取副本和copy类似这里感觉是Map过程，join合并和Reduce类似。
public static ExecutorService newWorkStealingPool() {
  		//创建一个ForkJoin的线程池。获取可用处理器（CPU）的个数，为parallelism，获取线程创建工厂。handler，和模式是异步还是同步。
        return new ForkJoinPool
            (Runtime.getRuntime().availableProcessors(),
             ForkJoinPool.defaultForkJoinWorkerThreadFactory,
             null, true);
    }
```

- unconfigurableExecutorService

```java
//创建一个不可代理的线程池。
public static ExecutorService unconfigurableExecutorService(ExecutorService executor) {
        if (executor == null)
            throw new NullPointerException();
  		//传入一个线程池作为参数，然后包装一下，只暴露ExecutorService相关的接口，其他接口被隐藏不能被设置。
        return new DelegatedExecutorService(executor);
    }
```

##### 线程池使用的注意问题

1、建议使用有界队列，有界队列能增加系统的稳定性和预警能力，防止资源过度消耗，撑爆内存，使得系统崩溃不可用。
2、提交到线程池的task之间要尽量保证相互独立，不能存在相互依赖，否则可能会造成死锁等其他影响线程池执行的原因。
3、提交到的线程池的task不要又创建一个子线程执行别的任务，然后又将这个子线程任务提交到线程池，这样会造成混乱的依赖，最终导致线程池崩溃，最好将一个task用一个线程执行。

4、一般需要根据任务的类型来配置线程池大小：

如果是**CPU密集型任务**，就需要尽量压榨CPU，参考值可以设为 **Num(CPU+1)**

如果是**IO密集型任务**，参考值可以设置为*2Num(CPU)**

当然，这只是一个参考值，具体的设置还需要根据实际情况进行调整，比如**可以先将线程池大小设置为参考值**，再**观察任务运行情况和系统负载、资源利用率来进行适当调整**。

Java线程底层映射到操作系统原生线程，而且Java在windows和linux平台下，一个Java线程映射为一个内核线程，而内核线程和CPU物理核心数一样，所以Java线程和CPU核心是一对一的关系，将线程池的工作线程设置为与物理核心相等能做到真正的线程并发，如果设置线程数多于核心则会在核心线程之间不停的切换。

##### 参考

[Java线程池ThreadPoolExecutor源码分析](https://fangjian0423.github.io/2016/03/22/java-threadpool-analysis/)

[线程池-ThreadPoolExecute源码分析](https://www.jianshu.com/p/7b1546a3b85b)

