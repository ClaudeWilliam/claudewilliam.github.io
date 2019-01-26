---
title: 浅析Java ArrayList源码
date: 2018-03-20 09:21:08
tags: [java,ArrayList]
categories: 源代码
---

##### 什么是ArrayList

Array是数组，List是线性表两个合起来就是一个数组化的线性表。也就是ArrayList是一个数组实现的列表。所以它有List的很多方法，可以实现List的功能，区别与数组ArrayList是可以自动扩容的。ArrayList的默认大小是10。（这个我也很好奇，为什么不是2的n次方的这种形式，后面有一个解释），每次扩容的时候是1.5倍，也不是两倍。同样ArrayList也是线程不安全的，也是用fast-fail机制。如果使用线程安全的类使用Vector，它使用了锁的同步机制实现了线程安全。但是效率比较低。

- ArrayList的继承关系图

![](/img/ArrayList-01.png)

##### ArrayList是如何实现的

先看一波ArrayList的定义的变量，从定义的变量中我们能看到，ArrayList的底层是基于一个数组实现的，它的默认大小是10，同时她存放数据的对象是不支持序列化的。

##### 常用的参数变量

````java
    
	//ArrayList实现了序列化的Serializable接口，这个是用于序列化的版本号。
    private static final long serialVersionUID = 8683452581122892189L;

    //默认的初始化大小
    private static final int DEFAULT_CAPACITY = 10;

    //用于空实例的共享空数组实例
    private static final Object[] EMPTY_ELEMENTDATA = {};

    //用于默认大小空实例的共享空数组实例。 我们将此与EMPTY_ELEMENTDATA区分开来，以了解什么时候第一个元素被添加
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    //用于存储ArryList元素的数组。ArrayList的容量就是数组的大小(这里不像HashMap有负载因子)， 任何用
	//elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA清空ArrayList在添加第一个元素时将扩展为DEFAULT_CAPACITY。
    transient Object[] elementData; // 这个对象不私有化是为了简化内部类的访问（而且在HashMap中这中容器也不是私有的）
	//数组的最大数量
	private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

	//ArrayList的大小，也就是有多少个元素
    private int size;
````

##### 构造函数

```java
//指定ArrayList的大小的构造函数。  
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {//如果参数大于0，创建initialCapacity大小的数组
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {//initialCapacity为0创建一个默认空数组
            this.elementData = EMPTY_ELEMENTDATA;
        } else {//如果initialCapacity<0抛出异常，非法参数
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }


 //构造一个初始容量为10的空列表。
 public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;//默认的一个空列表
    }

 //根据一个集合类对象，构造一个线性表
 public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();//把集合类对象转为数组，存放发到elementData数组中
        if ((size = elementData.length) != 0) {//如果原来集合对象长度不为0
           
            if (elementData.getClass() != Object[].class)//如果elementData的类型和要加入的数据类型不一致，返回不正确的object数组，重新拷贝一份。
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {//如果原来的对象为空，使用空的数组代替
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }


```

ArrayList的常用方法，通过这方法，实现了对线性表的最小维护。

##### 添加方法

```java
  public boolean add(E e) {

        ensureCapacityInternal(size + 1);  // Increments modCount!! 增加操作的记录数

        elementData[size++] = e;//给数组赋值，使用新add的对象

        return true;//返回结果为true

    }

 //向指定位置添加元素
 public void add(int index, E element) {

        rangeCheckForAdd(index);//判断添加是否数组越界

        ensureCapacityInternal(size + 1);  // Increments modCount!!增加操作的记录数

        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);//数组copy一下，原来的对象到+1的位置
        elementData[index] = element;//赋值

        size++;//大小+1

    }

//私有方法，用于判断minCapacity是否是最小的容量，minCapacity是最小需要的容量
 private void ensureCapacityInternal(int minCapacity) {

   		//如果elementData是默认的数组，就从默认值和最小容量选一个最小值
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
		//调用确保容量扩容的方法
        ensureExplicitCapacity(minCapacity);

    }

//私有方法，判断是否需要扩容
private void ensureExplicitCapacity(int minCapacity) {
        modCount++;//操作记录变量+1
        // overflow-conscious code 可能会溢出的代码
        if (minCapacity - elementData.length > 0)//如果最小容量大于现在数组容量，那么扩容，否则什么都不做。

            grow(minCapacity);//调用扩容方法。

    }


 

  //判断数组大小是否越界	
  private void rangeCheck(int index) {

        if (index >= size)//如果越界抛出异常
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

    }

  //设置指定位置的对象，并返回原来位置的对象
  public E set(int index, E element) {

        rangeCheck(index);//判断是否越界

        E oldValue = elementData(index);//获取之前的值

        elementData[index] = element;//设置新的值

        return oldValue;//返回旧的值

    }


 //添加时候的越界检验
 private void rangeCheckForAdd(int index) {

        if (index > size || index < 0)//如果指定位置大于数组大小，或者指定位置小于0                         
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));//抛出异常
    }

//根绝当前线性表的容量，去修剪数组（列表）的大小.一个应用可以使用这个操作最小化存储
 public void trimToSize() {

        modCount++;//fail-fast的标志，表示修改次数。这变量继承自AbstractList类

        if (size < elementData.length) {//如果当前容量大于当前数组（列表）的大小，进行修减，否则什么都不做

            elementData = (size == 0)//如果等于0，赋值空数组

            ? EMPTY_ELEMENTDATA: Arrays.copyOf(elementData, size);//重新copy一份新的

        }

    }


```

##### 扩容方法

```java
 //扩容方法。参数最小容量
 private void grow(int minCapacity) {

        // overflow-conscious code可能会溢出的代码
        int oldCapacity = elementData.length;//获取之前为未扩容的的数组大小

	   //根据未扩容的，计算新的大小，大小是原来的1.5倍。oldCapacity+0.5*oldCapacity，通过右移一位的方式除2
   		int newCapacity = oldCapacity + (oldCapacity >> 1);

   	   //如果新容量大于最最小容量，把新容量赋值给minCapacity
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)//如果新的容量大于最大数组大小
            newCapacity = hugeCapacity(minCapacity);//如果超出了调用方法来处理。
        // 根据新的容量和数组的大小，重新进行拷贝
        elementData = Arrays.copyOf(elementData, newCapacity);

    }

  //对超出容量进行处理。如果minCapacity<0说明溢出了，如果最小值大于最大容量，返回最大容量
  private static int hugeCapacity(int minCapacity) {

        if (minCapacity < 0) // overflow

            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE : MAX_ARRAY_SIZE;

    }

```

##### get方法和其他获取元素的方法

```java
  //根据对象在线性表的位置获取对象
   public E get(int index) {

        rangeCheck(index);//判断是否越界

        return elementData(index);//返回数组中的对象

    }
//获取数组对象的位置，从前往后找
public int indexOf(Object o) {
        if (o == null) {//如果对象为null那么查找为null的值
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;//返回结果
        } else {//for循环去找到位置
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;//返回-1没有找到
    }
//获取数组对象的位置，从后往前找
public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {//for循环去找到位置
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;//返回-1没有找到
    }
```

##### remove相关方法

```java
//删除节点，返回节之前节点的对象
public E remove(int index) {

        rangeCheck(index);//越界查询

        modCount++;//修改操作+1

        E oldValue = elementData(index);//获取之前节点的值

        int numMoved = size - index - 1;//获取删除节点到数组末尾的距离

        if (numMoved > 0)//如果大于0说明是在最后一个元素前面，进行数组copy
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);

        elementData[--size] = null; // clear to let GC do its work，否则删除最后一个节点

        return oldValue;//返回之前的对象

    }

 //移除指定对象
 public boolean remove(Object o) {

        if (o == null) {//如果对象为null，循环找到对象，删除后返回true
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {

                    fastRemove(index);
                    return true;
                }
        } else {//如果不为null，循环找出对象，删除返回true。

            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }

        }//如果上面都没走到，返回false
        return false;

    }

	//私有方法，跳过了边界检查，不会返回被移除的值
     private void fastRemove(int index) {
        modCount++;//修改次数+1
        int numMoved = size - index - 1;//获取要移动的位置

        if (numMoved > 0)//注释同上

            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

    }

	//清除数组里的所有信息
    public void clear() {

        modCount++;//修改次数+1
        // clear to let GC do its work，清空整个数组，让GC工作
        for (int i = 0; i < size; i++)

            elementData[i] = null;

		//修改size大小
        size = 0;

    }


 //移除集合中所有的对象
 public boolean removeAll(Collection<?> c) {
        Objects.requireNonNull(c);//判断是否为空，如果为空抛出异常
        return batchRemove(c, false);//调用批量移除方法。
    }
//批量的删除collection里面的元素。
private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;//获取当前线性表的元素
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)//循环列表中的所有数据进行比较，如果complement为true，将集合c中的元素，存放到数组的前面。W是从0开始
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {//如果r不等于size，对数组进行copy，复制r后面的元素
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {//如果w不等于size，循环删除下标w后的元素
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```

##### System中的copy方法

```java
//会经常用到System的arraycopy的native方法，了解arraycopy方法。参数src是源数组，srcPos是源数组的位置。dest是目标数组，destPos是目标数组的位置，从源数组到目标数组length是要复制的数组的长度。
 public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
```

##### 总结

ArrayList是一种常用集合类，是Java对数组的一种封装，同时它的泛型只支持包装类不支持基本类型。它不会像HashMap一样会有一个负载因子（比如0.75）。如果有的话也就1。也就是在ArrayList在被填满后自动扩容，而且它的扩容是1.5倍，不是2倍应该是基于一个空间上的考虑，毕竟一块连续的一块空间，尽量在使用的到的时候扩容，不能一次就扩容很大，对内存造成了浪费。

ArrayList和其他数组结构一样，都是查找的速度比较快O(1),但是插入和删除的速度，比较慢，要通过arraycopy效率相对比较低是O(n)，而链表恰恰相反，所以还要一种是LinkedList查找比较慢，但是插入和删除比较快但是查找比较慢。所以HashMap是一种比较好的设计，她结合了两者的有点。查找速度和删除速度都还不错，不过在空间消耗是要比List要好，所以总结一下还是**空间换时间**。所以在计算机中经常会有这种问题，好多数据结构一般都是比较浪费空间，来获取比较的时间，以上都是自己的拙见。

ArrayList这个源码也不是特别完整，包含她SubList的内部类没有介绍，和subList方法。

