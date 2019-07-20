---
title: dubbo的hessian序列化的坑
date: 2019-04-22 09:21:08
tags: [java,dubbo]
categories: dubbo
---

### 什么是序列化（serialization）

在计算机科学的数据处理中，是指将数据结构或对象状态转换成可取用格式（例如存成文件，存于缓冲，或经由网络中发送），以留待后续在相同或另一台计算机环境中，能恢复原先状态的过程。一般这个过程分两部分，一个是序列化，一个是反序列化。(这样说不便理解，我下面做一个简单的比喻)。

比如把一块肉和一本书，通过一个很细的管子传递到别的地方。（网络传输是字节流）

1、传肉的话要把肉切碎了传，到管子的另一边就成肉馅了。这个是不可被序列化的结果。

2、传书的话可以把书一页一页的撕下来卷成纸筒传过去，都传完之后按照页数排列好订在一起。这个是可被序列化的结果。（把书编页码，然后撕下来，这个是序列化过程。接收到在把书按照页数钉装在一起这个过程是反序列化）

一般Java中实现序列化都要去实现Serializable接口，同时也会有一个uuid。这个uuid一般都是自动生成的，这个主要用于反序列化时候区分版本的作用。

上面看上去那么麻烦，实现序列化，那为什么要实现序列化，实现序列化有什么好处。

### 为什么要序列化

先说好处，然后在解释。

1、序列化方便了网络数据的传输，可以通过序列化的方式，将数据进行压缩和分割，有利于网络传输，也保证了安全性。

2、序列化一般都可以跨平台，就是不同的语言的服务，只要实现了相同的序列化协议，那么久可以传输数据，不会限制语言。（java自带的序列化是自己玩的。但是hessian、json等等都是通用的）。

3、用于存储到硬盘，数据库中。一般我们说的对象，都是相对都是相对内存说的，但是存到数据库或者磁盘，是要进行序列化。（数据的持久化）

在网络传输中，一般都是tcp/ip协议。那也就是两台机器的两个进程在通信。我们平时对象或者数据结构都是根据自己本地的内存。那如何能让两台机器都能知道这个对象（数据结构）是什么样。就是通过序列化协议来约定。这样保证了数据准确性，同时在传输的过程中，他会进行压缩比如(Integer和int会转化成 i )这样可以减小传输的大小。同时保证了安全性，这样不容易被解析（上面例子是字符串，还有关于二进制的转换，或者是字节流的转换）。

跨平台，上面是两个不同机器和进程进行交互，同样的解析数据的语言或者处理操作系统也可能不一样。但是遵从了序列化的协议以后，根据一个序列化文件，反序列化出的数据结构和对象是一样的。

存储硬盘和数据库的时候要序列化。对于一般IO操作，我们都要把对象流化（IO化，变成输出输出流，这个输入输出是针对的内存。InputStream 文件—>内存，OutStream 内存—>文件）。但是对对象读写操作进行流化会有问题。

引用对象的存储问题。举个例子来说：假如我有两个类，分别是A和B，B类中含有一个指向A类对象的引用，现在我们对两个类进行实例化{ A a = new A(); B b = new B(); }，这时在内存中实际上分配了两个空间，一个存储对象a，一个存储对象b，接下来我们想将它们写入到磁盘的一个文件中去，就在写入文件时出现了问题！因为对象b包含对对象a的引用，所以系统会自动的将a的数据复制一份到b中，这样的话当我们从文件中恢复对象时(也就是重新加载到内存中)时，内存分配了三个空间，而对象a同时在内存中存在两份，想一想后果吧，如果我想修改对象a的数据的话，那不是还要搜索它的每一份拷贝来达到对象数据的一致性。序列化的出现解决了这个问题。

### 序列化算法的逻辑

1.对你遇到的每一个对象(元数据，具体只某个对象，而不是类)引用都关联一个序列号；

2.对于每一个对象，当第一次遇到时，保存其对象数据到流中。

3.如果某个对象之前已经保存过，那么在写的时候，只会写出与之前保存过的序列号相同的对象；

4.对于流中的对象，在第一次遇到其序列号时，构建它，并使用流中数据来初始化它。然后记录这个序列号和新对象之间的关联；

5.当遇到与之前保存过的序列号的对象相同时，获取这个顺序号相关联的对象引用；

关于1，2，3的解释

上面我们说到序列化不能存储引用对象，因为这样可能会有数据一致性的问题，肯定有人会想，可不可以存内存地址。这明显也是不可以的。首先内存是非持久化，内存地址会随时变调，所以在序列化的时候，一定要有东西看来标识对象。也就是说在序列化的中，对象引用都是通过序列号去区分的。

所以我们上面每一个对象都会用序列号去区分，然后根据序号去找对象，而这个对象，在第一次遇到的时候，就被保存到对象数据的流中。如果以后有其他对象引用了这个对象，那他会给一个序号，不是保存对象。如果有对象修改了，只修改存到对象数据流中。

关于4，5的解释 

4和5主要是反序列化的时候会用到。根据序列号去创建对象，然后使用流里的数据初始化他，同时把关联关系做好。把字节流的数据，转换成内存中的数据（有点像加载数据）。

某些数据域不需要进行序列，如只对本地方法有意义的存储文件句柄和窗口句柄的整数值。还有些域不想让它们序列化。这时可以用transient修饰这些域。（一般这些值都是，操作系统或者本地方法特有的值，比如说线程的状态变量）

对于不可序列化的属性，也需要用transient修饰，否则序列化时会抛出异常。

serialVersionUID的作用：

serialVersionUID 主要的作用就是版本控制。序列号操作的时候系统会把当前类的serialVersionUID写入到序列化流输出流中，当反序列化时系统会检查流中的serialVersionUID是否和当前类的serialVersionUID一致。如果一致，就说明可以反序列化，否则readObject会抛出InvalidClassException异常。

serialVersionUID生成：

可以不显示地指定这个常量，那么序列号化时jvm会自动生成一个版本号（根据当前类的信息，如类名、属性等）。这样每当类改变时，就不能兼容之前版本的已序列化的对象了；

如果在类中人工地指定了序列号private static final long serialVersionUID。则不会再自动生成，那么不同版本的类之间会尽量进行兼容，不会抛出InvalidClassException。如果流中的属性比当前的类多，则忽略多余的。如果当前类的属性比流中的多，则多余的赋成默认值。

一般在开发的过程中编译器，都会提示你加上serialVersionUID（如果你实现序列化接口）。这样可以防止不同jvm成版本的序列号不一致，导致的反序列化报错的问题。

### Java序列化和Hessian序列化区别

Java序列化：

　　Java序列化会把要序列化的对象类的元数据和业务数据全部序列化为字节流，而且是把整个继承关系上的东西全部序列化了。它序列化出来的字节流是对那个对象结构到内容的完全描述，包含所有的信息，因此效率较低而且字节流比较大。但是由于确实是序列化了所有内容，所以可以说什么都可以传输，因此也更可用和可靠。

hession序列化：

　　它的实现机制是着重于数据，附带简单的类型信息的方法。就像`Integer a = 1`，hessian会序列化成I 1这样的流，I表示int or Integer，1就是数据内容。而对于复杂对象，通过Java的反射机制，hessian把对象所有的属性当成一个Map来序列化，产生类似`M className propertyName1 I 1 propertyName S stringValue`（大概如此，确切的忘了）这样的流，包含了基本的类型描述和数据内容。而在序列化过程中，如果一个对象之前出现过，hessian会直接插入一个R index这样的块来表示一个引用位置，从而省去再次序列化和反序列化的时间。这样做的代价就是hessian需要对不同的类型进行不同的处理（因此hessian直接偷懒不支持short），而且遇到某些特殊对象还要做特殊的处理（比如StackTraceElement）。而且同时因为并没有深入到实现内部去进行序列化，所以在某些场合会发生一定的不一致，比如通过`Collections.synchronizedMap`得到的map。

### hessian序列化的实现和中间的问题

从上面我们知道，Java序列化，是基于对象和数据做全部序列化。这边保证的可靠和可用，但是效率低下(全部转换成字节)。而hessian是更偏向数据把简单数据类型压缩，同时通过map存储引用类型对象，然后通过反射的方式去实现对象的创建，这样效率高但是反射创建对象就会出写问题，特别是对继承相关的类在反序列化就会出现问题。

比如我之前碰到，就是在dubbo调用中，发现hessian反序列化的坑。（dubbo默认是使用hessian2序列化）

1、当用hessian序列化对象是一个对象继承另外一个对象的时候，当一个属性在子类和有一个相同属性的时候，反序列化后子类属性总是为null。

下面我们分析一下hessian序列化和反序列化的部分源码

```java
//hessian序列化源码，序列化中用到了很多反射的东西，它是把一个类拆成了数组（属性的数组）
public JavaSerializer(Class cl) {
    
     _writeReplace = getWriteReplace(cl);
    if (_writeReplace != null) _writeReplace.setAccessible(true);
    ArrayList primitiveFields = new ArrayList();//原始字段List
    ArrayList compoundFields = new ArrayList(); //复合字段
    //这里是两个for循环。将类中字段（Field）全部拿出来
    for (; cl != null; cl = cl.getSuperclass()) {//一直循环，直到获取当前类父类为null为止
      Field []fields = cl.getDeclaredFields(); //获取当前类所有字段
      for (int i = 0; i < fields.length; i++) { //将当前类的所有字段都遍历一遍
        Field field = fields[i]; 
      if (Modifier.isTransient(field.getModifiers())
        || Modifier.isStatic(field.getModifiers()))
      continue;  //如果当前字段不要序列化则跳过
    // XXX: could parameterize the handler to only deal with public
      field.setAccessible(true); //设置属性的访问权限，对于private类型的
      if (field.getType().isPrimitive()
        || (field.getType().getName().startsWith("java.lang.")
        && ! field.getType().equals(Object.class)))  
      primitiveFields.add(field); //判断字段是不是原始字段
    else
      compoundFields.add(field);
      }
    }
    ArrayList fields = new ArrayList();//创建一个属性List
    fields.addAll(primitiveFields);//加入原始属性
    fields.addAll(compoundFields);//加入符合属性
    _fields = new Field[fields.size()];//创建一个数组
    fields.toArray(_fields);//List转数组
  } //数组中是可以重复的元素存在的，所以父类和子类中的拥有相同属性的都会被保存下来。不被覆盖
```

反序列化的代码

```java
//反序列化的代码，获取个对象
protected HashMap<String,FieldDeserializer> getFieldMap(Class cl){
    //string 是的字段的名称 FieldDeserializer 存储的数据
    HashMap<String,FieldDeserializer> fieldMap //注意这里是一个Map，map的key是不可以重复的
     = new HashMap<String,FieldDeserializer>();
    for (; cl != null; cl = cl.getSuperclass()) {    //一直循环，直到获取当前类父类为null为止
      Field []fields = cl.getDeclaredFields();//获取当前类所有字段
      for (int i = 0; i < fields.length; i++) {//将当前类的所有字段都遍历一遍
        Field field = fields[i];
        if (Modifier.isTransient(field.getModifiers())
            || Modifier.isStatic(field.getModifiers()))
          continue; //如果 statuc和不需要序列化的对象跳过
        else if (fieldMap.get(field.getName()) != null)
          continue;
        // XXX: could parameterize the handler to only deal with public
        try {
          field.setAccessible(true); //设置访问属性
        } catch (Throwable e) {
          e.printStackTrace();
        }
        Class<?> type = field.getType(); //获取字段的class属性
        FieldDeserializer deser;

        if (String.class.equals(type))
          deser = new StringFieldDeserializer(field);  //如果是字符串字段，对字符串字段做反序列化处理
        else {
          deser = new ObjectFieldDeserializer(field); //否者认为是对象字段，做反序列化出口i
        }
        //就是在这里，如果后面有父类的字段名，子类的字段名就会被覆盖掉，
        fieldMap.put(field.getName(), deser); //字段名，具体字段是什么类型和值
      }
    }
    return fieldMap;
  }  
```

简言之，获取当前class的所有字段，接着获取父类的所有字段。序列化的时候，所有字段都放在一个ArrayList里，然后依次写入到二进制流中，反序列化的时候，所有字段放在了一个HashMap里，HashMap的key不能重复，悲剧就出现了，如果子类和父类有同名的字段就会有问题，父类的值会把子类的值覆盖掉。

2、参数及返回值不能自定义实现List, Map, Number, Date, Calendar等接口，只能用JDK自带的实现，因为hessian会做特殊处理，自定义实现类中的属性值都会丢失。

```java
@Override
    public Map getImmutableMap() {
        ImmutableMap<String, String> map = ImmutableMap.of("name", "roc", "sex", "boy");
        return map;
    }
//返回值
{
    "keys": [
        "name",
        "sex"
    ],
    "values": [
        "roc",
        "boy"
    ]
}
```

### 总结

关于序列化的小问题。

1、父类实现序列化，子类会自动实现序列化。但子类实现序列化父类不会。

2、一般序列化是，要把数据持久化，或者方便网络传输。（内存数据->字节/字符数据）

3、java自带的序列化，性能比较差，但是相对准确。hessain这种相对性能高一点，压缩了好多信息，所以在反序列化的时候会存在一些问题。

4、dubbo默认的序列化是hessian2的协议。

5、使用dubbo的一些约定：

参数及返回值需实现Serializable接口； 参数及返回值需有无参构造函数(可以是private的)或者有参构造所有函数允许传入null值。 参数及返回值不能自定义实现List, Map, Number, Date, Calendar等接口，只能用JDK自带的实现，因为hessian会做特殊处理，自定义实现类中的属性值都会丢失。

6、    Hessian序列化，只传成员属性值和值的类型，不传方法或静态变量，兼容情况：
    数据通讯     情况     结果
    A->B     类A多一种 属性（或者说类B少一种 属性）     不抛异常，A多的那 个属性的值，B没有， 其他正常
    A->B     枚举A多一种 枚举（或者说B少一种 枚举），A使用多 出来的枚举进行传输     抛异常
    A->B     枚举A多一种 枚举（或者说B少一种 枚举），A不使用 多出来的枚举进行传输     不抛异常，B正常接 收数据
    A->B     A和B的属性 名相同，但类型不相同     抛异常
    A->B     serialId 不相同     正常传输

  总结：会抛异常的情况：枚 举值一边多一种，一边少一种，正好使用了差别的那种，或者属性名相同，类型不同  

### 杂记

Java的原始属性（Primitive Field）：

```java
java.lang.Boolean#TYPE
java.lang.Character#TYPE
java.lang.Byte#TYPE
java.lang.Short#TYPE
java.lang.Integer#TYPE
java.lang.Long#TYPE
java.lang.Float#TYPE     
java.lang.Double#TYPE     
java.lang.Void#TYPE
```

在dubbo使用中注意些问题：

Dubbo缺省协议采用单一长连接和NIO异步通讯，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况。

缺省协议，使用基于netty3.2.2+hessian3.2.1交互。

​    连接个数：单连接
​    连接方式：长连接
​    传输协议：TCP
​    传输方式：NIO异步传输
​    序列化：Hessian二进制序列化
​    适用范围：传入传出参数数据包较小（建议小于100K），消费者比提供者个数多，单一消费者无法压满提供者，尽量不要用dubbo协议传输大文件或超大字符串。
​    适用场景：常规远程服务方法调用

为什么要消费者比提供者个数多：
因dubbo协议采用单一长连接，
假设网络为千兆网卡(1024Mbit=128MByte)，
根据测试经验数据每条连接最多只能压满7MByte(不同的环境可能不一样，供参考)，
理论上1个服务提供者需要20个服务消费者才能压满网卡。

为什么不能传大包：
因dubbo协议采用单一长连接，
如果每次请求的数据包大小为500KByte，假设网络为千兆网卡(1024Mbit=128MByte)，每条连接最大7MByte(不同的环境可能不一样，供参考)，
单个服务提供者的TPS(每秒处理事务数)最大为：128MByte / 500KByte = 262。
单个消费者调用单个服务提供者的TPS(每秒处理事务数)最大为：7MByte / 500KByte = 14。
如果能接受，可以考虑使用，否则网络将成为瓶颈。

为什么采用异步单一长连接：
因为服务的现状大都是服务提供者少，通常只有几台机器，
而服务的消费者多，可能整个网站都在访问该服务，
比如Morgan的提供者只有6台提供者，却有上百台消费者，每天有1.5亿次调用，
如果采用常规的hessian服务，服务提供者很容易就被压跨，
通过单一连接，保证单一消费者不会压死提供者，
长连接，减少连接握手验证等，
并使用异步IO，复用线程池，防止C10K问题。  

### 参考:

[Dubbo使用的注意事项](https://www.cnblogs.com/lishijia/p/5492847.html)

[Dubbo服务化改造时遇到的坑](https://www.fakemark.cn/dubbo-1510367570193.html)