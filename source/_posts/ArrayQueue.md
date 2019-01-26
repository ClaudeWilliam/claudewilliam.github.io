---
title: ArrayBlockingQueue
date: 2017-10-25 20:03:14
tags: [java,编程,源代码]
categories: java
---

**什么是Queue**

Queue就是队列的意思，所谓的队列就是排队，先进先出（FIFO first in frist out）。队列是一种非常常见的数据结构，他是一种特殊的线性表，只允许从head头出队，从tail尾入队。先进先出（FIFO）：先插入的队列的元素也最先出队列，类似于排队的功能。从某种程度上来说这种队列也体现了一种公平性。

**什么是Array**

在计算机科学中，数组数据结构（英语：array data structure），简称数组（英语：Array），是由相同类型的元素（element）的集合所组成的数据结构，分配一块**连续的内存来存储**。利用元素的索引（index）可以计算出该元素对应的存储地址。所以查找起来比较方便，和数组相对数据结构的就是链表，链表是不连续的存储。连续存储的好处就是她查找起来比较方便，每个元素都是相同大小的存放到一起。但是很容易出现碎片化的问题，而且对于大数组来说内存的消耗很大。但是链表就不会，他是很多和节点联系在一起，所以插入和删除比较方便，不用去移动位置。需要更改前驱和后继的指针就好了。

**什么是BlockingQueue**

Blocking（阻塞）Queue（队列）。多线程环境中，通过队列可以很容易实现数据共享，比如经典的“生产者”和“消费者”模型中，通过队列可以很便利地实现两者之间的数据共享。假设我们有若干生产者线程，另外又有若干个消费者线程。如果生产者线程需要把准备好的数据共享给消费者线程，利用队列的方式来传递数据，就可以很方便地解决他们之间的数据共享问题。但如果生产者和消费者在某个时间段内，万一发生数据处理速度不匹配的情况呢？理想情况下，如果生产者产出数据的速度大于消费者消费的速度，并且当生产出来的数据累积到一定程度的时候，那么生产者必须暂停等待一下（阻塞生产者线程），以便等待消费者线程把累积的数据处理完毕，反之亦然。（在多线程领域：所谓阻塞，在某些情况下会挂起线程（即阻塞），一旦条件满足，被挂起的线程又会自动被唤醒）

**当队列中没有数据的情况下，消费者端的所有线程都会被自动阻塞（挂起），直到有数据放入队列。**

**当队列中填满数据的情况下，生产者端的所有线程都会被自动阻塞（挂起），直到队列中有空的位置，线程被自动唤醒。**

**什么是ArrayBlockingQueue**

ArrayBlockingQueue就是通过数组来实现阻塞队列的一种方式。在ArrayBlockingQueue内部，维护了一个定长数组，以便缓存队列中的数据对象，这是一个常用的阻塞队列，除了一个定长数组外，ArrayBlockingQueue内部还保存着两个整形变量，分别标识着队列的头部和尾部在数组中的位置。

ArrayBlockingQueue在生产者放入数据和消费者获取数据，都是共用同一个锁对象，由此也意味着两者无法真正并行运行，这点尤其不同于LinkedBlockingQueue；按照实现原理来分析，ArrayBlockingQueue完全可以采用分离锁，从而实现生产者和消费者操作的完全并行运行。Doug Lea之所以没这样去做，也许是因为ArrayBlockingQueue的数据写入和获取操作已经足够轻巧，以至于引入独立的锁机制，除了给代码带来额外的复杂性外，其在性能上完全占不到任何便宜。 ArrayBlockingQueue和LinkedBlockingQueue间还有一个明显的不同之处在于，前者在插入或删除元素时不会产生或销毁任何额外的对象实例，而后者则会生成一个额外的Node对象。这在长时间内需要高效并发地处理大批量数据的系统中，其对于GC的影响还是存在一定的区别。而在创建ArrayBlockingQueue时，我们还可以控制对象的内部锁是否采用公平锁，默认采用非公平锁。

```java
//ArrayBlockingQueue 继承AbstractQueue实现了BlockingQueue接口和序列化接口
public class ArrayBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
  
  //这个Object数组就是用来存放ArrayBlockingQueue里面的对象。这里面也就ArrayBlockingQueue的Array
  /** The queued items */
   final Object[] items;
  
   //用于取出（take），删除（remove），出队（poll，peek）的对象的下标（索引）
   /** items index for next take, poll, peek or remove */
    int takeIndex;
    //用于放入（put），提供（offer），添加（add）的对象下标
  	/** items index for next put, offer, or add */
    int putIndex;
  	//用于记录队列里元素的个数
 	/** Number of elements in the queue */
    int count;
  
    /*
     * Concurrency control uses the classic two-condition algorithm
     * found in any textbook.
     * 并发控制使用任何教科书中的经典双条件算法。
     */
  
  	//守护所有访问的主要锁，也就是所有的访问都要通过这个锁
 	/** Main lock guarding all access */
    final ReentrantLock lock;
	
  	//取操作等待的条件（take，remove，poll）
    /** Condition for waiting takes */
    private final Condition notEmpty;
	
  	//放入操作的等待条件（put，offer，add）
    /** Condition for waiting puts */
    private final Condition notFull;
  
  
  /**
     * Shared state for currently active iterators, or null if there
     * are known not to be any.  Allows queue operations to update
     * iterator state.
     */
  
 // 当前活动迭代器的共享状态，如果不存在任何已知操作，则为null。 允许队列操作更新迭代器状态。
    transient Itrs itrs = null;
  
  
    /**
     * Circularly decrement i.从i循环递减到0
     */
    final int dec(int i) {
        return ((i == 0) ? items.length : i) - 1;
    }
    
   /**
     * Returns item at index i. 返回items i的元素
     */
    @SuppressWarnings("unchecked")
    final E itemAt(int i) {
        return (E) items[i];
    }
   /**
     * Throws NullPointerException if argument is null.
     *  校验参数v是否为空 v是元素
     * @param v the element
     */
    private static void checkNotNull(Object v) {
        if (v == null)
            throw new NullPointerException();
    }

  /**
     * Inserts element at current put position, advances, and signals.
     * Call only when holding lock.
     */
  //插入元素在put位置（下标putIndex位置），也就是当前插入下标的数组节点，同时唤醒持有锁的插入线程
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
      	//获取存储所有数据的数组
        final Object[] items = this.items;
      	//把数据放入当前下标节点的数组节点中
        items[putIndex] = x;
      	//判断一下是不是到数组最后节点也就 length-1，这里用的是++putIndex来说明
        if (++putIndex == items.length)
          	//如果是的话，就从头开始存放，因为那边消费也就是从投开始消费。
          	//即每次插入都从0开始，消费也都从0开始那么就可以实现先入先出
            putIndex = 0;
      	//加入成功后增加对列中元素数量，数组中元素加一
        count++;
      //唤醒那些等待插入的线程（持有锁的线程），可以插入。获取的Condition notEmpty的插入线程
        notEmpty.signal();
    }
    /**
     * Extracts element at current take position, advances, and signals.
     * Call only when holding lock.
     */
  //提取take位置的元素（下标takeIndex的元素），同时唤醒持有锁的取出线程
    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
      	//获取存储所有数据的数组
        final Object[] items = this.items;
      	//获取数组中take位置的元素
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
      	//将take位置制为空
        items[takeIndex] = null;
      	//判读是不是取数取到item数组的最后元素，如果超过会出现数组越界
        if (++takeIndex == items.length)
         //如果是从投开始取
            takeIndex = 0;
      	//对列中存在的元素数量减一
        count--;
      	//如果迭代器不为空，说明有线程把数据取走，元素减一
        if (itrs != null)
            itrs.elementDequeued();
      //唤醒取出线程（持有锁的）
        notFull.signal();
      //返回take节点元素
        return x;
    }
  
  
  /**
     * Deletes item at array index removeIndex.
     * Utility for remove(Object) and iterator.remove.
     * Call only when holding lock.
     */
  //删除item数组中index下标的元素，同时移除迭代器中数据，唤醒阻塞的线程
  //删除元素，不影响队列的顺序，就是要不从队列的前面删除，出队删除，也就是找到takeIndex，然后删除。
  //如果不是，那么就从队列最后面删除，即把之前的元素，和其他元素依次移动，然后把要删除的元素，移动到
  //putIndex哪里然后删除
    void removeAt(final int removeIndex) {
        // assert lock.getHoldCount() == 1;
        // assert items[removeIndex] != null;
        // assert removeIndex >= 0 && removeIndex < items.length;
      	//获取存储数据的数组
        final Object[] items = this.items;
      //如果取出位置，和要移除的位置正好是同一个（从队列前删除）
        if (removeIndex == takeIndex) {
            // removing front item; just advance
          	//清空位置信息
            items[takeIndex] = null;
          	//如果取出位置的下一个位置是最大位置，将取出位设成0，起始位置
            if (++takeIndex == items.length)
                takeIndex = 0;
          //count-- 表示位置减少
            count--;
          	//迭代器中的如果也存有值，那么将这个值也清除掉
            if (itrs != null)
                itrs.elementDequeued();
        } else {
            // an "interior" remove

            // slide over all others up through putIndex.
          //如果要删除位置，不是取出位置，那么进行循环，直到找到取出位置
            final int putIndex = this.putIndex;
            for (int i = removeIndex;;) {
              	//进行for的死循环
                int next = i + 1;
              	//如果next是数组的长度，那么从数组的长度为0也就是头部开始
                if (next == items.length)
                    next = 0;
              	//
                if (next != putIndex) {
                  //找到要移除的元素，然后把要移除的元素，放到putIndex哪里，然后删除掉（从队列后删除）
                    items[i] = items[next];
                    i = next;
                } else {
                  //删除队列，并设置putIndex值
                    items[i] = null;
                    this.putIndex = i;
                    break;
                }
            }
          //队列中的值减少
            count--;
          	//如果迭代器不为空，清除迭代器中存放的值
            if (itrs != null)
                itrs.removedAt(removeIndex);
        }
      //唤醒线程
        notFull.signal();
      
      /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity and default access policy.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity < 1}
     */
     //ArrayBlockingQueue 构造函数，默认是不公平锁，这里是指定了队列的大小
      /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity and the specified access policy.
     *
     * @param capacity the capacity of this queue
     * @param fair if {@code true} then queue accesses for threads blocked
     *        on insertion or removal, are processed in FIFO order;
     *        if {@code false} the access order is unspecified.
     * @throws IllegalArgumentException if {@code capacity < 1}
     */
    // ArrayBlockingQueue的构造函数，指定队列大小，和是否使用公平锁（一次插入，一次移除）
    public ArrayBlockingQueue(int capacity, boolean fair) {
      	//当 capacity小于0的时候抛出异常
        if (capacity <= 0)
            throw new IllegalArgumentException();
      	//创建大小合适的数组
        this.items = new Object[capacity];
      	//创建并发时候用的锁
        lock = new ReentrantLock(fair);
      	//入队条件
        notEmpty = lock.newCondition();
      	//出队条件
        notFull =  lock.newCondition();
    }
    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }
      
    }
  
  
    /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity, the specified access policy and initially containing the
     * elements of the given collection,
     * added in traversal order of the collection's iterator.
     *
     * @param capacity the capacity of this queue
     * @param fair if {@code true} then queue accesses for threads blocked
     *        on insertion or removal, are processed in FIFO order;
     *        if {@code false} the access order is unspecified.
     * @param c the collection of elements to initially contain
     * @throws IllegalArgumentException if {@code capacity} is less than
     *         {@code c.size()}, or less than 1.
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
  
  //将一个已有的集合，放入到队列中去
    public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
      	//初始化构造函数
        this(capacity, fair);
		//初始化锁
        final ReentrantLock lock = this.lock;
      	//将队列锁住，也就当前线程获取锁。
        lock.lock(); // Lock only for visibility, not mutual exclusion
      	//然后将集合中的值遍历出来放到队列数组中去
        try {
            int i = 0;
            try {
                for (E e : c) {
                  	//校验是否为空，是空则抛出空指针异常
                    checkNotNull(e);
                    items[i++] = e;
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                throw new IllegalArgumentException();
            }
            count = i;
          	//初始化队尾，如果等于最大值，为变为头，如果不是则为 i
            putIndex = (i == capacity) ? 0 : i;
        } finally {
          	//释放锁
            lock.unlock();
        }
    }
  
  
   /**
     * Inserts the specified element at the tail of this queue if it is
     * possible to do so immediately without exceeding the queue's capacity,
     * returning {@code true} upon success and throwing an
     * {@code IllegalStateException} if this queue is full.
     *
     * @param e the element to add
     * @return {@code true} (as specified by {@link Collection#add})
     * @throws IllegalStateException if this queue is full
     * @throws NullPointerException if the specified element is null
     */
   //向队列加入值，这里面调用的是父类的方法，父类中调用的是offer()方法，
  //如果添加成功返回true，失败则抛出异常（也就是）队列满了情况下
    public boolean add(E e) {
        return super.add(e);
    }
  
  
  /**
     * Inserts the specified element at the tail of this queue if it is
     * possible to do so immediately without exceeding the queue's capacity,
     * returning {@code true} upon success and {@code false} if this queue
     * is full.  This method is generally preferable to method {@link #add},
     * which can fail to insert an element only by throwing an exception.
     *
     * @throws NullPointerException if the specified element is null
     */
  //插入一个元素，向队列中，如果插入成功返回true，失败返回false
  
    public boolean offer(E e) {
      	//如果插入的值是null，抛空指针异常
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
      	//当前线程获取锁，也就是操作权限
        lock.lock();
        try {
          	//队列已满，插入失败
            if (count == items.length)
                return false;
            else {
              //插入对列，返回成功
                enqueue(e);
                return true;
            }
        } finally {
          //释放锁
            lock.unlock();
        }
    }
 
  //出队
   public E poll() {
     	//获取到锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
          //如果count等于0返回null，否则出队一个元素
            return (count == 0) ? null : dequeue();
        } finally {
          //最后释放锁
            lock.unlock();
        }
    }
  
  	//从队列取出一个元素
    public E take() throws InterruptedException {
      //获取锁
        final ReentrantLock lock = this.lock;
      	//如果当前线程没有被打断，获取锁。如果这个 锁没有被获取
        lock.lockInterruptibly();
        try {
          	//当count==0也就是队列是空的情况，线程一直保持等待，其他线程进不来
            while (count == 0)
                notEmpty.await();
          	//如果不是空则，出队一个元素
            return dequeue();
        } finally {
          //释放锁
            lock.unlock();
        }
    }
  
  //在一段时间出队一个元素，时间是纳秒
   public E poll(long timeout, TimeUnit unit) throws InterruptedException {
     	//使用TimeUnit设置时间长度
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
       //如果当前线程没有被打断，获取锁。如果这个 锁没有被获取
        lock.lockInterruptibly();
        try {
          	//如果队列元素为0个，同时超过了等待时间，那就返回一个空，否则就继续等待
            while (count == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
          	//如果队列元素不是0，那就出队一个元素
            return dequeue();
        } finally {
          	//释放锁
            lock.unlock();
        }
    }
  
  	//从队列头取出一个元素
    public E peek() {
      	//获取锁，如果没有获取，那么线程就一直处于等待状态
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
          //返回一个元素，即takeIndex位置的元素。都是最前面的元素
            return itemAt(takeIndex); // null when queue is empty
        } finally {
          //释放锁
            lock.unlock();
        }
    }
// this doc comment is overridden to remove the reference to collections
  //这个文章的注解@overridden，被移除掉引用从集合接口中
    // greater in size than Integer.MAX_VALUE
    /**
     * Returns the number of elements in this queue.
     * 返回队列元素的个数
     * @return the number of elements in this queue
     */
    public int size() {
      	//获取锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
          //返回count值，即队列中元素的数量
            return count;
        } finally {
          //释放锁
            lock.unlock();
        }
    }
  
  // this doc comment is a modified copy of the inherited doc comment,
    // without the reference to unlimited queues.
    /**
     * Returns the number of additional elements that this queue can ideally
     * (in the absence of memory or resource constraints) accept without
     * blocking. This is always equal to the initial capacity of this queue
     * less the current {@code size} of this queue.
     *
     * <p>Note that you <em>cannot</em> always tell if an attempt to insert
     * an element will succeed by inspecting {@code remainingCapacity}
     * because it may be the case that another thread is about to
     * insert or remove an element.
     */ 
  //队列中剩余多少可以插入的元素，一般就是用数组的总长度减去当前元素的个数
    public int remainingCapacity() {
      	//获取锁，如果没有获取等待
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
          	//返回当前线程时，剩余能插入的元素个数
            return items.length - count;
        } finally {
          	//释放锁
            lock.unlock();
        }
    }
  
   /**
     * Removes a single instance of the specified element from this queue,
     * if it is present.  More formally, removes an element {@code e} such
     * that {@code o.equals(e)}, if this queue contains one or more such
     * elements.
     * Returns {@code true} if this queue contained the specified element
     * (or equivalently, if this queue changed as a result of the call).
     *
     * <p>Removal of interior elements in circular array based queues
     * is an intrinsically slow and disruptive operation, so should
     * be undertaken only in exceptional circumstances, ideally
     * only when the queue is known not to be accessible by other
     * threads.
     *
     * @param o element to be removed from this queue, if present
     * @return {@code true} if this queue changed as a result of the call
     */
  	//从队列里删除一个元素o
    public boolean remove(Object o) {
      	//如果o是null返回false
        if (o == null) return false;
      	//获取存取队列的到当前的数组
        final Object[] items = this.items;
      	//获取锁，如果获取不到进入等待，等待condition.signal()唤醒
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
          	//如果队列中有元素，就去查找移除，否则返回false
            if (count > 0) {
                final int putIndex = this.putIndex;
                int i = takeIndex;
              	//通过do while循环去遍历整个数组，也就是队列。从对列的头进入。也就是从takeIndex节点开始
                do {
                  	//如果找到，然后调用removeAt将元素删除，并且调整队列
                    if (o.equals(items[i])) {
                        removeAt(i);
                      //返回成功
                        return true;
                    }
                  	//如果达到数组的最大长度，然后从数组的头开始，即第一个元素
                    if (++i == items.length)
                        i = 0;
                  //当i等于putIndex也就是从头找到尾，因为putIndex即队尾
                } while (i != putIndex);
            }
          	//否则返回false，就是没找到相等的对象，在这个队列
            return false;
        } finally {
          	//释放锁
            lock.unlock();
        }
    }

  
   /**
     * Returns {@code true} if this queue contains the specified element.
     * More formally, returns {@code true} if and only if this queue contains
     * at least one element {@code e} such that {@code o.equals(e)}.
     *
     * @param o object to be checked for containment in this queue
     * @return {@code true} if this queue contains the specified element
     */
  	//是否包含当前元素o
    public boolean contains(Object o) {
      	//如果当前元素是null返回false
        if (o == null) return false;
      	//获取当前数组存储队列值
        final Object[] items = this.items;
      	//获取锁，如果获取不到进入等待，等待condition.signal()唤醒
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
          	//如果队列里有元素就去查找，否则返回false
            if (count > 0) {
              	//获取当前队列中队列尾部在数组中位置
                final int putIndex = this.putIndex;
              	//获取当前队列的开头部分，也就是出队位置
                int i = takeIndex;
                do {
                  	//如果队列开头位置的元素，是包含的，返回true
                    if (o.equals(items[i]))
                        return true;
                   //如果达到数组的最大长度，然后从数组的头开始，即第一个元素
                    if (++i == items.length)
                        i = 0;
                  //循环到队列尾部，结束也就是i==putIndex
                } while (i != putIndex);
            }
            return false;
        } finally {
          //释放锁
            lock.unlock();
        }
    }

  
   /**
     * Atomically removes all of the elements from this queue.
     * The queue will be empty after this call returns.
     */
  //移除该队列中所有元素
    public void clear() {
      	//获取存放队列元素的数组
        final Object[] items = this.items;
      //获取锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
          	//获取队列张还有多少元素
            int k = count;
          	//如果元素小于等于0，说明没有元素队列
            if (k > 0) {
              	//获取队尾的节点的数组位置
                final int putIndex = this.putIndex;
              	//获取队头节点的数组位置
                int i = takeIndex;
                do {
                  	//清空该节点的位置，也就是对头元素位置
                    items[i] = null;
                  	//是否达到数组最大边界,如果是从数组第一个元素开始
                    if (++i == items.length)
                        i = 0;
                //直到把整个队列数据清空  
                } while (i != putIndex);
              	//清空后把队尾，和队列头放到一起
                takeIndex = putIndex;
              	//清空队列里计算
                count = 0;
              	//清空迭代器里的值，告诉迭代器现在队列中值是空的
                if (itrs != null)
                    itrs.queueIsEmpty();
              	//如果之前队列元素，那么看一下还有多少个等待添加的线程，如果有唤醒他们，让他们向队列添加。也就是
              	//clear队列，对应的是当前线程，如果有加入线程，在当前线程清空后还可以加入。
                for (; k > 0 && lock.hasWaiters(notFull); k--)
                    notFull.signal();
            }
        } finally {
          	//释放锁
            lock.unlock();
        }
    }
  //最多从此队列中移除给定数量的可用元素，并将这些元素添加到给定 collection 中 。返回值int代表添加了多少个 
  public int drainTo(Collection<? super E> c, int maxElements) {
    	//校验给定的集合是不是空，如果是空抛出空指针异常
        checkNotNull(c);
    	//如果这个给定的集合，等于当前这个队列，抛出非法参数异常
        if (c == this)
            throw new IllegalArgumentException();
    	//如果要移除的元素个数小于等于0，直接返回0
        if (maxElements <= 0)
            return 0;
    	//获取存放队列的数组
        final Object[] items = this.items;
    	//获取锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
          	//从要转移的最大元素个数，和线程中存在的元素个数取小的那个
            int n = Math.min(maxElements, count);
          	//获取队列的起始位置，也就是队列的头
            int take = takeIndex;
            int i = 0;
            try {
              	//开始循环，取出值
                while (i < n) {
                    @SuppressWarnings("unchecked")
                  //从队列取出值放到集合中去，然后删除队列中的值
                    E x = (E) items[take];
                    c.add(x);
                    items[take] = null;
                  	//如果队列头位置，到达数组最大值那么从数组第一元素开始
                    if (++take == items.length)
                        take = 0;
                  	//取出元素+1
                    i++;
                }
              	//最后返回n个
                return n;
            } finally {
                // Restore invariants even if c.add() threw
                if (i > 0) {
                  	//成功个数大于0,将队列中元素减去i个
                    count -= i;
                  	//设置takeIndex位置
                    takeIndex = take;
                  	//如果迭代器不等于null
                    if (itrs != null) {
                      	//如果count等于0，说明队列中没有元素
                        if (count == 0)
                          	//设置迭代器队列是空
                            itrs.queueIsEmpty();
                        else if (i > take)
                          	//否则用Index的值来覆盖到迭代器中
                            itrs.takeIndexWrapped();
                    }
                  	//唤醒等待的线程
                    for (; i > 0 && lock.hasWaiters(notFull); i--)
                        notFull.signal();
                }
            }
        } finally {
          	//释放锁
            lock.unlock();
        }
    	//这个有两个try操作中有两个finally，一个是处理迭代器的，另一个是处理锁的
    }
  
  
```

其实ArrayBockingQueue中还有几个内部类没有说，但是我在这里就不多解释，包括序列化和toString方法，迭代器方法和内部类没有介绍。所以我这也就不过多的介绍，本文是基于JDK 1.8.0_121的进行的分析。 ArrayBlockingQueue实现的队列中的锁是没有分离的，即生产和消费用的是同一个锁。而且ArrayBlockingQueue相比LinkedBlockingQueue性能更高。因为ArrayBlocking是通过数组，查找更快，移除使用过的元素会更快，但删除队列中元素对整个队列排序时候LinkedBlockingQueue更快。而且队列长度越大越明显，也会移动调整元素数量更多。但一般情况队列都不会存特别多数据。LinkedBlockingQueue最大长大度也就是Integer的最大值