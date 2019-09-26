---
title: Spring事务浅谈
date: 2019-09-22 09:21:08
tags: [java,spring]
categories: spring
---

### Spring事务浅谈

#### 什么是事务

简单点来说，就是一组被包装过的操作（一般是更新数据库，插入或者更新多个库的操作，一般读数据库不需要事务，都是在写操作的时候）。包装过的这些操作是可以回滚的。简单点说，就是这组操作中有一个或者失败，就全体失败回滚，只有全部成功才能才算执行完成。（比较大白话），**事务是一个不可分割操作序列，也是数据库并发控制的基本单位，其执行的结果必须使数据库从一种一致性状态变到另一种一致性状态。**一般事务会有四个属性（ACDI）原子性，一致性，隔离性，持久性。具体详细的内容我之前博客是有的详细的介绍。[事务](https://blog.51cloud.win/2017/06/17/%E4%BA%8B%E5%8A%A1/#more)

#### Spring的事务管理

上面说的事务，是事务的基本概念，一般事务多与数据库相关，比如mysql，jdbc等等都有对事务的支持。但是除了数据库，随着机器的不断增多，越来越多的分布式事务出现了。比如现在比较常用的方式就是使用事务的mq，来实现分布式事务的。也有什么两步提交，三步提交理论。我们不详细的展开了。这里我们重点是说Spring对事务的管理。**Spring没有事务，他只是抽象出对事务的管理接口`TransactionManager`，然后其他组件（jdbc）去实现spring的接口。让spring可以管理数据库相关的事务**。下面就是内部的事务管理器的类图。

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/TransactionManager.png)

Spring事务管理器的描述

| 事务管理器                       | 使用场景                                                     |
| -------------------------------- | ------------------------------------------------------------ |
| `DataSourceTransactionManager`   | 数据源事务管理器，提供对单个`javax.sql.DataSource`事务管理，用于Spring JDBC抽象框架、iBATIS或MyBatis框架的事务管理； |
| `JdoTransactionManager`          | 提供对单个`javax.jdo.PersistenceManagerFactor`y事务管理，用于集成JDO框架时的事务管理； |
| `JpaTransactionManager`          | 提供对单个`javax.persistence.EntityManagerFactory`事务支持，用于集成JPA实现框架时的事务管理； |
| `WebSphereUowTransactionManager` | Spring提供的对WebSphere 6.0+应用服务器事务管理器的适配器，此适配器用于对应用服务器提供的高级事务的支持； |
| `WebLogicJtaTransactionManager`  | Spring提供的对WebLogic 8.1+应用服务器事务管理器的适配器，此适配器用于对应用服务器提供的高级事务的支持。 |

在Spring中主要就是通过事务管理器来控制事务，只不过就是针对不同的框架做了不同的适配。 **所谓事务管理，实质上就是按照给定的事务规则来执行提交或者回滚操作**。

Spring事务管理有三个比较基本的类。

##### 1、定义事务操作的基本接口（与框架无关的操作）`PlatformTransactionManager`类

```java
package org.springframework.transaction;

public interface PlatformTransactionManager {
    //获取事务状态，包含事务这个只是一个接口
    TransactionStatus getTransaction(TransactionDefinition var1) throws TransactionException;
	//提交事务
    void commit(TransactionStatus var1) throws TransactionException;
	//回滚事务
    void rollback(TransactionStatus var1) throws TransactionException;
}
```

这里面`AbstractPlatformTransactionManager`这个类是抽象类，这个类主要是实现各种通用的方法。这种设计模式，一般叫模板方法。

##### 2、存储事务的状态的情况的基本接口`TransactionStatus`类

```java
package org.springframework.transaction;

import java.io.Flushable;

public interface TransactionStatus extends SavepointManager, Flushable {
    //返回当前交易是否为新事务。这个主要用于事务传播
    boolean isNewTransaction();
	//返回此事务是否在内部携带保存点，，即已基于保存点创建为嵌套事务
    boolean hasSavepoint();
	//设置事务仅回滚。这指示给事务管理器，事务的唯一可能结果可能是回滚，因为引发异常的方法，后者将触发回滚
    void setRollbackOnly();
	//返回事务是否已被标记为仅回滚
    boolean isRollbackOnly();
	//将session刷新到数据存储区：例如，所有受影响的Hibernate / JPA session
    void flush();
	//返回该事务是否完成，即是否已提交或回滚
    boolean isCompleted();
}

```

##### 3、关于事务的定义。定义了事务常用的常量。和获取事务属性获取方法

```java

package org.springframework.transaction;

public interface TransactionDefinition {
    //0-6传播策略
    //Support a current transaction; create a new one if none exists.
    int PROPAGATION_REQUIRED = 0;
    //Support a current transaction; execute non-transactionally if none exists.
    int PROPAGATION_SUPPORTS = 1;
    //Support a current transaction; throw an exception if no current transaction
    int PROPAGATION_MANDATORY = 2;
    //Create a new transaction, suspending the current transaction if one exists.
    int PROPAGATION_REQUIRES_NEW = 3;
    //Do not support a current transaction; rather always execute non-transactionall
    int PROPAGATION_NOT_SUPPORTED = 4;
    //Do not support a current transaction; throw an exception if a current transaction
    int PROPAGATION_NEVER = 5;
    // Execute within a nested transaction if a current transaction exists,
    int PROPAGATION_NESTED = 6;
    //事务隔离策略 1-4 8序列化
    //默认的事务隔离策略，一般mysql的数据库事务隔离策略是可重复读，mysql是使用多版本并发控制策略实现的。但是sqlServer，oracle都读已提交。
    int ISOLATION_DEFAULT = -1;
    //读未提交
    int ISOLATION_READ_UNCOMMITTED = 1;
    //读已提交
    int ISOLATION_READ_COMMITTED = 2;
    //可重复读
    int ISOLATION_REPEATABLE_READ = 4;
    //串行化，没有并发
    int ISOLATION_SERIALIZABLE = 8;
    //使用，事务系统默认的超时。没有超时
    int TIMEOUT_DEFAULT = -1;
	//获取传播策略
    int getPropagationBehavior();
	//获取隔离级别
    int getIsolationLevel();
	// 获取超时值
    int getTimeout();
	//是不是只读事务
    boolean isReadOnly();
	//获取事务名字
    String getName();
}

```

我们先简单的简介一下，我们这里主要是看接口中的方法，这些方法就是Spring平台抽象出来关于事务的操作，后面会详细介绍传播机制。和隔离策略。

#### Spring事务的5个属性

从上面三个接口中我们也能猜个差不多，关于Spring事务抽象出来的5属性。1、传播行为；2、隔离级别；3、回滚规则；4、事务超时；5、是否只读；

`@Transactional(readOnly = true,rollbackFor = Exception.class,propagation = Propagation.REQUIRED,value = "transactionManager",timeout = 10,isolation = Isolation.REPEATABLE_READ)`先写一个注解，后面我们会解释

##### 1、传播行为

**所谓事务的传播行为是指，如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。所以事务的传播一定是有两个以上不同的事务同时执行。且这两个事务会有相互影响。**（事务嵌套）这两个事务在一个线程。TransactionDefinition接口定义了如下几个表示传播行为的常量：

`PROPAGATION_REQUIRED`: 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。

`PROPAGATION_SUPPORTS`: 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。

`PROPAGATION_MANDATORY`:如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。

`PROPAGATION_REQUIRES_NEW`:创建一个新的事务，如果当前存在事务，则把当前事务挂起。

`PROPAGATION_NOT_SUPPORTED`: 以非事务方式运行，如果当前存在事务，则把当前事务挂起。

`PROPAGATION_NEVER`:以非事务方式运行，如果当前存在事务，则抛出异常。

`PROPAGATION_NESTED`:如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。（有内外事务才算嵌套，内部事务会依赖外部提交，外部事务提交才会提交。外部事务回滚内部事务也会回滚。内部事务不能单独的提交和回滚）。

从上面我们可以看出来，除了嵌套事务是同时执行，就没有两个事务同时执行（内部还要依赖外部）。都是只有一个事务运行，或者没有事务运行。

下面是`PROPAGATION_REQUIRES_NEW`和`PROPAGATION_NESTED`两种表现形式

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/PROPAGATION.png)

Spring默认是传播策略PROPAGATION_REQUIRED

`@Transactional(propagation = Propagation.REQUIRED)` 

##### 2、隔离级别

**所谓事务的隔离级别是指若干个并发的事务之间的隔离程度**。所以事务隔离是解决并发问题的。也就是当多个事务操作同一个热点数据的时候就会有数据一致性问题。一般专业数据就是脏读，幻读。简单点说就是读到数据，不是最终数据库持久化的数据。TransactionDefinition 接口中定义了五个表示隔离级别的常量。（也是数据库常见的）

`ISOLATION_READ_UNCOMMITTED`: RU 读未提交。

`ISOLATION_READ_COMMITTED`: RC  读以提交。

`ISOLATION_REPEATABLE_READ`: RR 可重复读。

`ISOLATION_SERIALIZABLE`: 串行化。

默认的事务隔离策略，一般mysql的数据库事务隔离策略是可重复读，mysql是使用多版本并发控制策略实现的。但是sqlServer，oracle都读已提交。

- 脏读(Drity Read)：**某个事务**已更新一份数据，**另一个事务**在此时读取了同一份数据，由于某些原因，前一个RollBack了操作，则后一个事务所读取的数据就会是不正确的。
- 不可重复读(Non-repeatable read):在**一个事务**的两次查询之中数据不一致，这可能是两次查询过程中间插入了一个事务更新的原有的数据。**即一次事务中不可已重复读取数据**
- 幻读(Phantom Read):在**一个事务**的两次查询中数据笔数不一致，例如有一个事务查询了几列(Row)数据，而**另一个事务**却在此时插入了新的几列数据，先前的事务在接下来的查询中，就会发现有几列数据是它先前所没有的。

![img](https://blog.51cloud.win/img/af5b9c1e-4517-3df2-ad62-af25d1672d12.jpg)

Spring默认是隔离级别 ISOLATION_REPEATABLE_READ

`@Transactional(isolation = Isolation.REPEATABLE_READ)` 

##### 3、回滚规则

**通常情况下，如果在事务中抛出了未检查异常（继承自 RuntimeException 的异常），则默认将回滚事务**。如果没有抛出任何异常，或者抛出了已检查异常，则仍然提交事务。这通常也是大多数开发者希望的处理方式，也是 EJB 中的默认处理方式。但是，我们可以根据需要人为控制事务在抛出某些未检查异常时任然提交事务，或者在抛出某些已检查异常时回滚事务。
所以在事务中很多的时候，都要指定要捕获的异常类型。一般可以被抛出来的类。`Throwable,Exception,Error,RuntimeException `这些类。一般会指定为Exception。Spring默认回滚类RuntimeException

`@Transactional(rollbackFor=Exception.class)` 

##### 4、事务超时

**所谓事务超时，就是指一个事务所允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务。**在 TransactionDefinition 中以 int 的值来表示超时时间，其单位是秒。Spring默认超时是-1.也就是没有超时。

`@Transactional(timeout=100)`

##### 5、是否只读

**事务的只读属性是指，对事务性资源进行只读操作或者是读写操作。**所谓事务性资源就是指那些被事务管理的资源，比如数据源、 JMS 资源，以及自定义的事务性资源等等。如果确定只对事务性资源进行只读操作，那么我们可以将事务标志为只读的，以提高事务处理的性能。在 TransactionDefinition接口中，以 boolean 类型来表示该事务是否只读。Spring中默认的只读是false

`@Transactional(readOnly=false)`

我们在最开始写的注解中，基本上所有值都使用，还有一个就是value。这个value是指定事务管理器。一般用于指定dataSource。一般用于多数据源时候去处理。

#### Spring 声明式事务管理

在Spring中大多数使用事务都是使用的声明式事务。**Spring 的声明式事务管理是建立在 Spring AOP 机制之上的，其本质是对目标方法前后进行拦截，并在目标方法开始之前创建或者加入一个事务，在执行完目标方法之后根据执行情况提交或者回滚事务。**

##### 开启事务

一般事务开启分两种方式，一种是通过注解，一种是配置xml。随着注解越来越流行，现在使用xml配置的也不多的了。但是本质上都是一样的。都是启用了spring中配置好的一个Bean。

```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
<property name="dataSource" ref="dataSource"/>
</bean>
```

上面这种方法和`DataSourceTransactionManager transactionManager= new  DataSourceTransactionManager(datasource)`效果是一样的。一般用于SpringMVC项目。现在微服务大行其道，大家都使用`@EnableTransactionManagement `这个注解启用事务管理机制。

##### 事务的使用

事务的使用是通过`@Transactional`注解。这个注解主要声明在类上或者方法上。如果声明在方法，表明这个方法是执行事务的。如果声明在类上，表示这个类所有的方法，都是事务方法。我们一般使用事务方法比较多。因为事务执行是很耗费性能。所以尽量减小事务的逻辑，一般只有update和insert才使用事务。这样有利于提高性能。

但是select也可以使用事务。使用select的事务可以保证select查询是走的主库而不是从库（在数据库读写分离的时候会，在主库插入和从库读取有比较大延时），一般会把select和insert或update放到一起去执行。不用去在sql上去写`/*master*/`这种去强制sql去走主库。也就是只有执行事务时去从主库拿。其他的时候都走从库。

同样的，`@Transactional`里面还有很可以配置的属性。我上面与介绍。你可以去根据自己业务需求去添加。

#### Spring事务使用的注意

![](https://raw.githubusercontent.com/ClaudeWilliam/pic-blog/master/img/image001.jpg)

上面我们说Spring的事务是基于AOP去做的。也就是Spring对原有类TargetClass进行动态代理。对原有的方法进行加强。

1、首先调用的拦截，先通过`CglibAopProxy$DynamicAdvisedInterceptor`进行拦截。然后执行`invokeWithinTransaction()`方法。来开始执行事务

2、事务拦截器`TransactionInterceptor`拦截到对应的事务，然后进行增强和执行事务，在执行方法。

3、通过`AbstractPlatformTransactionManager`去对事务的传播，事务的回滚规则做处理。

4、执行业务方法。

5、commit()或者rollback()提交给`DataSource`，让`DataSource`做处理。

##### @Transactional 只能应用到 public 方法才有效

只有`@Transactional `注解应用到 public 方法，才能进行事务管理。这是因为在使用 Spring AOP 代理时，Spring  中的 `TransactionInterceptor` 在目标方法执行前后进行拦截之前，`DynamicAdvisedInterceptor`（`CglibAopProxy` 的内部类）的的 intercept 方法或 `JdkDynamicAopProxy` 的 invoke 方法会间接调用 `AbstractFallbackTransactionAttributeSource`（Spring 通过这个类获取表 1. `@Transactional `注解的事务属性配置属性信息）的 `computeTransactionAttribute` 方法。

```java
protected TransactionAttribute computeTransactionAttribute(Method method,

    Class<?> targetClass) {

        // Don't allow no-public methods as required.

        if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {

return null;}
```

其实根本的原因，是AOP的cglib是生成代理类，把原有对象放到新的代理类中加强，但是私有方法不能在代理类中使用。所以也就不支持私有方法。

##### 避免 Spring 的 AOP 的自调用问题

在 Spring 的 AOP 代理下，只有目标方法由外部调用，目标方法才由 Spring 生成的代理对象来管理，这会造成自调用问题。若同一类中的其他没有@Transactional 注解的方法内部调用有`@Transactional` 注解的方法，有`@Transactional` 注解的方法的事务被忽略，不会发生回滚。

```java
public class A{
    
   @Transactional 
   public int createA(){
       
   }
     
   public void updateB(){
        createA();
    }

}
```

createA()中的事务不会被实现。因为执行事务很关键一点，就是被`CglibAopProxy$DynamicAdvisedInterceptor`拦截器拦截。但是在同一个类中这个拦截器是拦截补刀updateB()方法调用createA()方法。他属于**类内调用**，所以不会触发事务。这个也是其他情况下使用Aop要注意的一点。这个是可以通过使用 AspectJ 取代 Spring AOP 代理，来解决问题。

#### 总结

​	Spring是没有事务，他是通过抽象接口和方法和定义一些事务相关的操作和事务管理器。然后让DataSource或者JTA这种去实现他的抽象，实现了Spring对事务的管理。同样Spring的声明式事务也是基于AOP去实现的。通过动态代理的方式来增强Target类的属性。在事务中也有常见的设计模式，比如模板方法和策略模式。他抽象方法让不同的框架去实现不同的策略，这个和Java的SPI和像。有时间可以多了解Dubbo的好多组件都是通过SPI去实现的。

#### 参考

[Spring 事务管理机制概述](https://blog.csdn.net/justloveyou_/article/details/73733278)

[透彻的掌握 Spring 中@transactional 的使用](https://blog.csdn.net/qq_29347295/article/details/79019221)

