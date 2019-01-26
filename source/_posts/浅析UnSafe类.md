---
title: 浅析UnSafe
date: 2018-08-25 09:21:08
tags: [java,编程]
categories: 编程
---

#### UnSafe类

 UnSafe是一个类，它存放于`package sun.misc;`是Java的一个基本类。看这个包名应该是属于BootStrap加载器加载的类。其次他是一个`public final class Unsafe {}`Final的类，他不能被直接的new 出来而是通过反射的方式获取到（后面会有代码做演示）。同样它也不能被继承，所以一般情况下我们在做应用开发的时候很少会使用到这类，但是这个类却在很多框架和并发包中大量的使用，所以想深入的了解Java还是要知道一些关于UnSafe类的知识。

从这个类的名字Unsafe上来说这个类就是一个不安全的类，也是不开放给用户直接使用的（当然我们还是可以通过其他一些方法用到）。这个类在jdk源码中多个类中用到，主要作用是任意内存地址位置处读写数据，外加一下CAS操作。它的大部分操作都是绕过JVM通过JNI(Java Native Interface)完成的，因此它所分配的内存需要手动free（因为绕过了JVM所以就没有垃圾回收），所以是非常危险的（容易内存泄漏）。但是Unsafe中很多（但不是所有）方法都很有用，且有些情况下，除了使用JNI，没有其他方法弄够完成同样的事情。

#### 如何创建一个UnSafe对象

上面我们说UnSafe是一个final类，不能被new出来，那么我们怎么去创建这个对象，首先我们先看一下UnSafe的构造函数

```java
private static final Unsafe theUnsafe;//静态变量

private Unsafe() {
 }//私有化构造函数

 @CallerSensitive
 public static Unsafe getUnsafe() {//提供获取对象的唯一方法
 Class var0 = Reflection.getCallerClass();
   if(!VM.isSystemDomainLoader(var0.getClassLoader())) {//用于判断是不是系统的ClassLoader
     throw new SecurityException("Unsafe");//如果不是BootStrapClassLoader抛出异常 
   } else { //也就是默认不让其他对象去获取
     return theUnsafe;
   }
 }

//存放在VM工具类中方法如果这个对象是不存在JAVA堆中，都可以认为是系统的ClassLoader即BootStrapClassLoader
public static boolean isSystemDomainLoader(ClassLoader var0) {
        return var0 == null;
}
```

通过上面的代码，很明显是一个单例模式，所以我们只能反射去获取getUnsafe方法来获取对象的实例。而且判断一下是否是使用BootStrapClassLoader加载的Unsafe.Class。据说在JDK9中可以直接获取。而且在JDK9中UnSafe类也会由原来`package jdk.internal.misc`。也就UnSafe类被开放使用，同样也是使用的单例，不过在JDK9中原来的方法也没有删除，依然可以使用反射的方式得到。

```java
//JDK9中的源码 
private static native void registerNatives();
    static {
        registerNatives();
    }

    private Unsafe() {}

    private static final Unsafe theUnsafe = new Unsafe();
	public static Unsafe getUnsafe() {
        return theUnsafe;
    }
```

创建UnSafe对象有两种方式一种是通过反射。

```java
Field f = Unsafe.class.getDeclaredField("theUnsafe");//通过反射获取这个静态成员变量
f.setAccessible(true);//设置访问权限
Unsafe unsafe = (Unsafe) f.get(null);//获取Unsafe对象，因为是静态对象所以用null

```

或者通过JVM配置来获取。将你的类使用BootStrapClassLoader类加载器去加载。一般会使用`java -Xbootclasspath`这个参数。

```
-Xbootclasspath:完全取代系统Java classpath.最好不用。
-Xbootclasspath/a: 在系统class加载后加载。一般用这个。
-Xbootclasspath/p: 在系统class加载前加载,注意使用，和系统类冲突就不好了.
java -Xbootclasspath/a: some.jar:some2.jar:  -jar test.jar 
/usr/jdk1.7.0/jre/lib/rt.jar:. com.mishadoff.magic.UnsafeClient
```

上面这两种都可以一种是把你自己的类放入某个jar使用在系统class加载后加载，或者在com.mishadoff.magic.UnsafeClient类在rt.jar后加载，rt.jar肯定是用BootStrapClassLoader加载。不过一般都是使用反射的方式去获取没有人使用JVM配置去获取，毕竟这太复杂了。

#### UnSafe主要都能使用场景

##### 大数组

我们都知道Java的数组最大长度是Integer的最大值，使用直接内存分配我们可以创建非常大的数组，该数组的大小只受限于堆的大小。如果想做一个比这大的数组通过正常渠道是不可能的，所以我们要通过UnSafe这个对象去操作。

```java
package com.qjq;


import sun.misc.Unsafe;

import java.lang.reflect.Field;

public class SuperArray {
    private final static int BYTE = 1;

    private long size;
    private long address;

    public SuperArray(long size) {
        this.size = size;
       //申请内存的方法和C语言中的malloc类似，这里返回的是申请的地址
        address = getUnsafe().allocateMemory(size * BYTE);//一个字节大小的数组
    }

     public void set(long i, byte value) {
        getUnsafe().putByte(address + i * BYTE, value);//在申请的地址进行偏移，我们都知道数组是连续的一段内存
    }

      public int get(long idx) {
        return getUnsafe().getByte(address + idx * BYTE);//根据Id也就数组下标，计算出偏移量来找地址
    }

    public long size() {
        return size; //返回数组大小
    }

    public void free() {
        getUnsafe().freeMemory(address);//释放内存
    }

    private Unsafe getUnsafe() {
        Field f = null; //通过反射获取这个静态成员变量
        Unsafe unsafe = null;
        try {

            f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);//设置访问权限
            //获取Unsafe对象，因为是静态对象所以用null
            unsafe = (Unsafe) f.get(null);
        } catch (Exception e) {
            System.out.println(e);
        }
        return unsafe;
    }
}
//这个是个数组代码，这里使用的是JDK9的包可以直接使用UnSafe类，如果自己写的话可以参照上面去写获取UnSafe的方法
import static jdk.internal.misc.Unsafe.getUnsafe;

public class SuperArray {
    private final static int BYTE = 1;

    private long size;
    private long address;

    public SuperArray(long size) {
        this.size = size;
        //申请内存的方法和C语言中的malloc类似，这里返回的是申请的地址
        address = getUnsafe().allocateMemory(size * BYTE);//一个字节大小的数组
    }

    public void set(long i, byte value) {
        getUnsafe().putByte(address + i * BYTE, value);//在申请的地址进行偏移，我们都知道数组是连续的一段内存
    }

    public int get(long idx) {
        return getUnsafe().getByte(address + idx * BYTE);//根据Id也就数组下标，计算出偏移量来找地址
    }

    public long size() {
        return size; //返回数组大小
    }
}
```

一个简单的用法：

```java
public class MaxTest {
    public static void main(String[] args) {
        long sum=0L;
        long SUPER_SIZE = (long)Integer.MAX_VALUE * 2;
        SuperArray array = new SuperArray(SUPER_SIZE);
        System.out.println("Array size:" + array.size()); // 4294967294
        for (int i = 0; i < 100; i++) {
            array.set((long)Integer.MAX_VALUE + i, (byte)3);
            sum += array.get((long)Integer.MAX_VALUE + i);
        }
        System.out.println("Sum of 100 elements:" + sum);  // 300
        array.free();
    }
}
-----结果
Array size:4294967294
Sum of 100 elements:300


```

事实上该技术使用了非堆内存`off-heap memory`，在 `java.nio` 包中也有使用。NIO中的DirectByteBuffer也是通过UnSafe去操作的，一般可以指定：-XX:MaxDirectMemorySize=40M这个的大小。堆内在flush到远程时，会先复制到直接内存（非堆内存），然后在发送；而堆外内存相当于省略掉了这个工作，避免来回复制，估计了解Netty的人都知道。

通过这种方式分配的内存不在堆上，并且不受GC管理。因此需要小心使用`Unsafe.freeMemory(address)`。该方法不会做任何边界检查，因此任何不合法的访问可能就会导致JVM奔溃。

这种使用方式对于数学计算是非常有用的，因为代码可以操作非常大的数据数组。 同样的编写实时程序的程序员对此也非常感兴趣，因为不受GC限制，就不会因为GC导致非常大的停顿。所以从某个角度可以减少GC的压力。

##### 并发中的使用

在Java并发中会用到CAS操作，对应于Unsafe类中的compareAndSwapInt，compareAndSwapLong等，可以用来实现高性能的无锁化数据结构。

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates，这里主要使用compareAndSwapInt用于更新
    //Unsafe变量，AtomicInteger在rt.jar所以可以直接使用
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset
      
    static {
        try {
            valueOffset = unsafe.objectFieldOffset//获取value偏移量
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;//用于真正存储的数，这个值是volatile说明是内存可见
      
    //CAS方法
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
  	//先获取值，在自增一
   public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
  
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
}  
```

还有很多，比如ConcurrentHashMap，AQS等多线程的类中广泛用到这种原子操作。都是依赖UnSafe类来实现。主要都是CAS操作。这样效率更高，但是也会有ABA问题，不过这里就不讨论晚上有人说AtomicStampedReference/AtomicMarkableReferenc解决，具体没研究过。

##### sizeof方法

使用objectFieldOffset方法我们可以实现C语言风格的sizeof方法。下面的方法实现返回对象的表面上的大小。（也就是引用指向的大小）

```java
public static long sizeOf(Object o) {
    Unsafe u = getUnsafe();//获取UnSafe对象
    HashSet<Field> fields = new HashSet<Field>();//获取Set集合存有成员变量
    Class c = o.getClass();//根据对象获取类，这里是通过引用获取class。比如这一块是多态就就不样
    while (c != Object.class) {//进行遍历，除了Object对象。
        for (Field f : c.getDeclaredFields()) {//获取当前类的声明的变量
          //获取变量的修饰符的大小。与Static修饰符进行yu操作如果操作为0.说明没有static修饰
            if ((f.getModifiers() & Modifier.STATIC) == 0) {
                fields.add(f);//不计算对象中static变量的大小，因为static是类的属性。
            }
        }
        c = c.getSuperclass();//一直去找这个对象，除非他是object类，也就找到了头
    }

    // get offset
    long maxSize = 0;
    for (Field f : fields) {
        long offset = u.objectFieldOffset(f);//获取每个变量的偏移的地址
        if (offset > maxSize) {
            maxSize = offset;//取偏移值的最大值
        }
    }

    return ((maxSize/8) + 1) * 8;   // padding 根据最大偏移值计算对象的大小
}
```

##### 浅拷贝

同样使用上面的方法也可以实现浅copy。通过实现计算表面大小，我们可以简单地添加复制对象的功能。标准解决方案需要修改的代码Cloneable，或者你可以在对象中实现自定义复制功能，但它不是多用途功能。

```java
//浅copy
static Object shallowCopy(Object obj) {
    long size = sizeOf(obj);//对象大小
    long start = toAddress(obj);//起始地址,也就是对象转地址
    long address = getUnsafe().allocateMemory(size);//创建对象的地址
    getUnsafe().copyMemory(start, address, size);//将
    return fromAddress(address);//返回对象
}

static long toAddress(Object obj) {//对象转地址
    Object[] array = new Object[] {obj};
    long baseOffset = getUnsafe().arrayBaseOffset(Object[].class);
    return normalize(getUnsafe().getInt(array, baseOffset));
}

static Object fromAddress(long address) {
    Object[] array = new Object[] {null};
    long baseOffset = getUnsafe().arrayBaseOffset(Object[].class);
    getUnsafe().putLong(array, baseOffset, address);
    return array[0];
}

private static long normalize(int value) {
    if(value >= 0) return value;//如果值大于0，直接返回
    return (~0L >>> 32) & value;//取反后无符号右移32位
}

System.out.println(normalize(10));//10
System.out.println(normalize(-1));//4294967295
System.out.println(normalize(-10));//4294967286
System.out.println(normalize(Integer.MIN_VALUE));//2147483648
System.out.println(normalize(Integer.MAX_VALUE));//2147483647
System.out.println(~0L);//取反

```

此复制功能可用于复制任何类型的对象，其大小将动态计算。请注意，复制后需要将对象强制转换为特定类型。

##### 内存泄漏攻击（Memory  corruption）

内存邪路攻击对与C语言来说很常见，但是对于java程序员来说就比较少见，一般都是由JVM去操作内存。

```java
class Guard {
    private int ACCESS_ALLOWED = 1;

    public boolean giveAccess() {
        return 42 == ACCESS_ALLOWED;
    }
}
```

对于客户端来说这段代码是非常安全的，并且调用 `giveAccess()`检查访问规则。对于客户来说，它总会回归`false`。只有特权用户才能*以某种方式*更改常量值`ACCESS_ALLOWED`并获取访问权限。不幸的是，事实并非如此

```java
Guard guard = new Guard();// 创建对象
guard.giveAccess();   // false, no access  返回没有权限访问

// bypass
Unsafe unsafe = getUnsafe(); //获取UnSafe对象
Field f = guard.getClass().getDeclaredField("ACCESS_ALLOWED"); //获取ACCESS_ALLOWED成员变量
unsafe.putInt(guard, unsafe.objectFieldOffset(f), 42); // memory corruption 修改内存中的值

guard.giveAccess(); // true, access granted
```

##### 跳过构造初始化

`allocateInstance`当您需要跳过对象初始化阶段或绕过构造函数中的安全检查，或者您想要该类的实例但没有任何公共构造函数时，方法可能很有用。思考下面的例子

```java
class A {
    private long a; // not initialized value

    public A() {
        this.a = 1; // initialization
    }

    public long a() { return this.a; }
}
```

使用构造函数，反射和Unsafe实例化它会产生不同的结果。

```java
A o1 = new A(); // constructor
o1.a(); // prints 1

A o2 = A.class.newInstance(); // reflection
o2.a(); // prints 1

A o3 = (A) unsafe.allocateInstance(A.class); // unsafe
o3.a(); // prints 0
```

是想一下UnSafe用实例化，于单例模式的类会怎么样。

同样Unsafe类也有实现多重继承，抛出异常，快速序列化用法，这里就不一一展开了。下面我们去看一下UnSafe的函数。

#### UnSafe主要使用的函数（Java里说方法的比较多）

上面提到说UnSafe是根据JNI接口去实现了好多方法，所以在它的源码中大多数都是Native方法。从上的代码中我们Java的内存地址使用的是long这种长整型来表示。下面的代码主要是针对JDK8上去解释，JDK9中的代码实现和JDK8在实现上有很大的差别。毕竟从原来的一个UnSafe类现在变成了两个Unsafe类。`sun.misc`包里额Unsafe类主要是使用的是`jdk.internal.misc`中的对象。

- 返回一个低级别的内存相关的信息

  addressSize
  pageSize

- 提供操作对象和对象字段的方法

  allocateInstance
  objectFieldOffset

- 提供针对类和类的静态字段操作的方法

  staticFieldOffset
  defineClass
  defineAnonymousClass
  ensureClassInitialized

- 数组操作

  arrayBaseOffset
  arrayIndexScale

- 低级别的同步原语（JDK8中好多都不推荐使用，JDK9中被移除）

  monitorEnter（移除）
  tryMonitorEnter（移除）
  monitorExit（移除）
  compareAndSwapInt 
  putOrderedInt

- 直接访问内存的方法

  allocateMemory
  copyMemory
  freeMemory
  getAddress
  getInt
  putInt

```java
//返回一个静态字段的偏移量
public native long staticFieldOffset(Field f);

//返回一个非静态字段字段的偏移量
public native long objectFieldOffset(Field f);

@HotSpotIntrinsicCandidate //JDK9中会有这个注解
public native int getInt(Object o, long offset);//根据给定对象指定的一个偏移量获取Int值

//把给定对象指定偏移量上的值设置为整型变量ｘ
public native void putInt(Object o, long offset, int x);

//获取给定内存地址上的byte值
public native byte   getByte(long address);

//把给定内存地址上的值设置为byte值ｘ
public native void  putByte(long address, byte x);

//获取给定内存地址上的值，也就是指针
public native long getAddress(long address);

//分配一块本地内存，这块内存的大小便是给定的大小．这块内存的值是没有被初始化的，返回值是一个地址
public native long allocateMemory(long bytes);
//扩充内存  
public native long reallocateMemory(long address, long bytes);        
//释放内存参数是内存地址
public native void freeMemory(long address);  
//在给定的内存块中设置值，object对象，offset是指定偏移量，bytes是设置大小，value的值
public native void setMemory(Object o, long offset, long bytes, byte value); 

//取消阻塞线程 thread,如果 thread 已经是非阻塞状态，
//那么下次对该 thread 进行park操作时就不会阻塞
public native void unpark(Object thread);

//阻塞当前线程，isAbsolute
//为true时，表示绝对时间，单位是毫秒   time = System.currentTimeMillis() + 3000  表示当前时间3秒后
//为false时，表示相对时间，单位是纳秒   time = 30000000，00挂起一段时间
//park(false, 0)表示一直等待，直到有unpark(thisThread)（如果该方法调用前在线程非阻塞情况下调用了unpart方法，那么此次调用该方法不会令当前线程阻塞）
public native void park(boolean isAbsolute, long time);

//先获取在自增  var1给定的对象，va2指定对象上指定的偏移量，一般是指某个成员变量，var4就是要增加的值
public final int getAndAddInt(Object var1, long var2, int var4) {
  int var5;//原来的值
  do {
    var5 = this.getIntVolatile(var1, var2);//获取当前的值
    //死循环直到替换完成，此时cpu一直在空转也可以说是自旋
  } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

  return var5;//替换完成后返回值
}
//现获取在设置int值  var1给定的对象，va2指定对象上指定的偏移量，一般是指某个成员变量，var4就是要设置的值
public final int getAndSetInt(Object var1, long var2, int var4) {
  int var5;//原来的值
  do {
    var5 = this.getIntVolatile(var1, var2);
      //死循环直到替换完成，此时cpu一直在空转也可以说是自旋
  } while(!this.compareAndSwapInt(var1, var2, var5, var4));

  return var5;//替换完成后返回值
}

//CAS，如果对象偏移量上的值=期待值，更新为x,返回true.否则false.类似的有compareAndSwapInt,compareAndSwapLong,compareAndSwapBoolean,compareAndSwapChar等等。  
public final native boolean compareAndSwapObject(Object o, long offset,  Object expected, Object x);  
// 该方法获取对象中offset偏移地址对应的整型field的值,支持volatile load语义。类似的方法有getIntVolatile，getBooleanVolatile等等  
public native Object getObjectVolatile(Object o, long offset); 

//创建一个类的实例，不需要调用它的构造函数、初使化代码、各种JVM安全检查以及其它的一些底层的东西。即使构造函数是私有，我们也可以通过这个方法创建它的实例。上面有提到
public native Object allocateInstance(Class cls) throws InstantiationException;

//抛出异常，没有通知 也就是throw new Throwable;
public native void throwException(Throwable e);

 //可以获取数组第一个元素的偏移地址,也就是数组的地址
public native int arrayBaseOffset(Class arrayClass);  
//可以获取数组的转换因子，也就是数组中元素的增量地址。将arrayBaseOffset与arrayIndexScale配合使用， 可以定位数组中每个元素在内存中的位置  
public native int arrayIndexScale(Class arrayClass);  
//获取本机内存的页数，这个值永远都是2的幂次方  
public native int pageSize();  
//告诉虚拟机定义了一个没有安全检查的类，默认情况下这个类加载器和保护域来着调用者类  
 public native Class defineClass(String name, byte[] b, int off, int len, ClassLoader loader, ProtectionDomain protectionDomain);  
//定义一个类，但是不让它知道类加载器和系统字典  
public native Class defineAnonymousClass(Class hostClass, byte[] data, Object[] cpPatches);
//内存屏障，禁止load操作重排序，即屏障前的load操作不能被重排序到屏障后，屏障后的load操作不能被重排序到屏障前
public native void loadFence();
//内存屏障，禁止store操作重排序，即屏障前的store操作不能被重排序到屏障后，屏障后的store操作不能被重排序到屏障前
public native void storeFence();
//内存屏障，禁止load、store操作重排序
public native void fullFence();
```

#### 总结

Java是一个安全的编程语言，它能最大程度的防止程序员犯一些低级的错误（大部分是和内存管理有关的）。但凡是不是绝对的，使用Unsafe程序员就可以操作内存，因此可能带来一个安全隐患。但是同时UnSafe类的使用也让Java可以像C和C++一样去直接操纵底层的内存，这样可以提高效率。也方便了好多操作为好多框架的实现和使用成为了可能，比如Netty（堆外内存的分配基于UnSafe），Kryo的快速序列化等等。

Unsafe这个名字也告诉我们这个类是不安全的类，我们一般在应用开发很少用到这个类，如果使用这个类要记住UnSafe类的操作的内存不是JVM堆中的内存，他不受垃圾回收的影响，所以要记得手动释放，否则会造成严重的内存泄漏问题。

了解UnSafe类还是好的，他可以让你更了解计算机的底层，了解JVM之外的事情。

#### 杂记

@CallSensitive是JVM中专用的注解，在类加载过过程中是可以常常看到这个注解的身影的。

当通过反射调用时，必须专门处理这些方法，以确保getCallerClass方法返回实际调用者的类，而不是反射机制本身的某些类。 这样做的逻辑涉及模式匹配手动维护的调用者敏感方法列表。[可能处于安全的考虑](https://blog.csdn.net/HEL_WOR/article/details/50199797)

JAVA 反射机制中，Field的getModifiers()方法返回int类型值表示该字段的修饰符。其中，该修饰符是java.lang.reflect.Modifier的静态属性。

对应表如下：PUBLIC: 1，PRIVATE: 2，PROTECTED: 4，STATIC: 8，FINAL: 16，SYNCHRONIZED: 32，VOLATILE: 64，TRANSIENT: 128，NATIVE: 256，INTERFACE: 512，ABSTRACT: 1024，STRICT: 2048。

而反射中获取getModifiers()方法是把各个修饰符的值相加。

```java
    private int age; //2

    public String address; //1 

    public static String name; //9

    public static final String world="world"; //25

    public  final  String hello="hello"; //17    
```

@HotSpotIntrinsicCandidate 注解

学过java编译过程的都知道编译会进行热点代码的优化，如：方法内联、常量传播、空值检查消除、寄存器分配等等，热点代码一般通过热点探测得出，而HotSpotIntrinsicCandidate注解能够直接手段将某个方法直接指定为热点代码，jvm尽快优化它（非绝对优化，优化时机不确定）。

```java
//可以参考的小demo
public class UnsafeCASTest {
    public static void main(String[] args) throws Exception {
        // 通过反射实例化Unsafe
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        Unsafe unsafe = (Unsafe) f.get(null);

        // 实例化Player
        Player player = (Player) unsafe.allocateInstance(Player.class);
        player.setAge(18);
        player.setName("li lei");
        for (Field field : Player.class.getDeclaredFields()) {
            System.out.println(field.getName() + ":对应的内存偏移地址" + unsafe.objectFieldOffset(field));
        }

        System.out.println("-------------------");
        // unsafe.compareAndSwapInt(arg0, arg1, arg2, arg3)
        // arg0, arg1, arg2, arg3 分别是目标对象实例，目标对象属性偏移量，当前预期值，要设的值

        int ageOffset = 12;
        // 修改内存偏移地址为12的值（age）,返回true,说明通过内存偏移地址修改age的值成功
        System.out.println(unsafe.compareAndSwapInt(player, ageOffset, 18, 20));
        System.out.println("age修改后的值：" + player.getAge());
        System.out.println("-------------------");

        // 修改内存偏移地址为12的值，但是修改后不保证立马能被其他的线程看到。
        unsafe.putOrderedInt(player, 12, 33);
        System.out.println("age修改后的值：" + player.getAge());
        System.out.println("-------------------");

        // 修改内存偏移地址为16的值，volatile修饰，修改能立马对其他线程可见
        unsafe.putObjectVolatile(player, 16, "han mei");
        System.out.println("name修改后的值：" + unsafe.getObjectVolatile(player, 16));
    }
}

class Player {
    private int age;
    private String name;
    private Player() {
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }

}
```



参考： [ Java Magic. Part 4: sun.misc.Unsafe](http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/)