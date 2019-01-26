---
title: 函数式编程与Java的Lambda
date: 2018-10-25 09:21:08
tags: [java,编程]
categories: 编程
---

#### 什么是函数式编程

##### 什么是函数式编程

**函数式编程**（functional programming）：或称函数程序设计，又称泛函编程，是一种编程范式，它将计算机运算视为数学上的函数计算，并且避免使用程序状态以及易变对象。函数编程语言最重要的基础是λ演算（lambda calculus）。而且λ演算的函数可以接受函数当作输入（引数）和输出（传出值）。

比起指令式编程，函数式编程更加强调程序执行的结果而非执行的过程，倡导利用若干简单的执行单元让计算结果不断渐进，逐层推导复杂的运算，而不是设计一个复杂的执行过程。函数式编程是偏结构化的编程。

如果看到上面你感觉没懂，或者一脸懵逼。那就对了，上面是非常抽象的，书面的表达；相对来说不是那么容易理解。所以我们要换个方法一点一点的带入。首先从上面提到一个指令式编程，根据这个对比我们看起来会相对容易理解。指令式编程又叫命令式编程，我们一般接触这种类型的语言多一点，比如说C，C++，JAVA很多语言都是命令式编程。我们现在引入命令式编程的概念。

##### 什么是命令式编程

**命令式编程(Imperative programming）：**是一种描述计算机所需作出的行为的编程范式。机器语言及汇编语言是最早的命令式语言。在这种语言中，（冯诺依曼机）计算机被看做是动作的序列，程序就是用语言提供的操作命令书写的一个操作序列（程序代码）。用命令式程序设计语言编写程序，就是描述解题过程中每一步的过程，程序的运行过程就是问题的求解过程（因此也称为过程式语言）。

大多数命令式编程有相似的风格，比如说他们都有功能类似的语句。赋值语句，运算语句，循环语句，条件分支语句，和无条件分支语句。

赋值语句：就是常规的赋值，把数据存放一个存储器中或者是建立联系（指针，引用）。在C，C++中就是`a=2;`这种语句。

运算语句：一般来说都表现了在存储器内的数据进行运算的行为，然后将结果存入存储器中以便日后使用。`a=b+3;`

循环语句：一些语句反复运行数次，可以根据条件。`while(a>0) {a--;}`或者是`for(i=0;i<10;i++)`

条件分支语句：仅当某些条件成立时才运行某个区块。`if(a>0){....}else{.....}`

无条件分支语句：运行顺序转移到程序的其他部分之中。`goto语句 `

说了这么多，也就是我们平时使用的C，C++，JAVA语言都是命令式编程，就是把一些逻辑和操作，抽象成一个个命令，然后输入到计算机，计算机依次执行（没有并发的情况下）他会完成我们要进行的运算。（也可以说我们处理问题，要把自己当成计算机去想问题，告诉计算机如何去做，每一步都干什么，都要写到代码里）而我们写的指令也就是代码（准确说是写完编译好的二进制文件）。这种想法来自冯诺依曼（大神），主要应用在冯诺依曼机（以冯诺依曼原型为基础的计算机）。简单介绍一下冯诺依曼机，一般冯诺依曼机是由控制器，运算器，内存，输入设备，输出设备五部分构成。也是现代计算机的原型（祖先）。

![](/img/func/function-01.jpg)

函数式编程和上面的逻辑思维是不一样的，在命令式编程上，要把你要做的事情要抽象成指令，然后一步一步的去操作。执行效率取决于执行指令数量。但是在函数式编程上，就不是这样，在函数式编程中没有什么指令，只有函数，函数是这类编程语言的一等公民。强调将计算过程分解成可复用的函数，典型例子就是`map`方法和`reduce`方法组合而成MapReduce。只有纯的、没有副作用的函数，才是合格的函数。

- 函数是一等公民，指的是函数与其他数据类型一样，处于平等地位，可以赋值给其他变量，也可以作为参数，传入另一个函数，或者作为别的函数的返回值。

```javascript
var print = function(i){ console.log(i);};
```

- 在计算的过程中，是使用函数（计算过程分解成可复用的函数），而不是各种语句，比如说循环语句和判断语句，是我们在命令式编程中经常使用的。而函数一般都会有运算过程，入参和出参（常见的递归）。而语句是没有运算的，多是控制流程。


- 没有副作用的函数，即那些没有全局变量的函数。也就是函数的计算只会影响函数内部，而不会影响到其他函数。也就没有什么公共状态。（所以一般函数式编程咋多线程下都是安全的）同时函数的返回值不会依赖系统的变量或者变化。这样就会使函数式编程更加安全。

但是无论是函数式编程还是命令式编程，他们现在的底层实现基本上都是基于执行指令来完成任务。因为现代计算机都是基于冯诺依曼机去实现的。（命令式编程基于的是冯诺依曼机，函数式是基于图灵机）

##### 简单总结一下

一切都是数学函数。函数式编程语言里也可以有对象，但通常这些对象都是恒定不变的 —— 要么是函数参数，要什么是函数返回值。函数式编程语言里没有 for/next 循环，因为这些逻辑意味着有状态的改变。相替代的是，这种循环逻辑在函数式编程语言里是通过递归、把函数当成参数传递的方式实现的。

举个例子：a = a + 1

这段代码在普通成员看来并没有什么问题，但在数学家看来确实不成立的，因为它意味着变量值得改变。

#### 什么是lambda演算

lambda演算是由阿隆佐-邱奇提出的。lambda是一个希腊的符号λ。

λ演算（英语：lambda calculus，λ-calculus）是一套从数学逻辑中发展，以变量绑定和替换的规则，来研究**函数如何抽象化定义、函数如何被应用以及递归**的形式系统。（来自wiki中文，但是感觉听上去不像人话）

发明Lambda演算的作者叫做阿隆佐邱奇（Alonzo Church）， 发明图灵机的作者叫做阿兰图灵（Alan Turing）。他们几乎活跃在同一个时代，他们那个时代的数学界有个领袖，叫希尔伯特（David Hilbert， 德国数学家），当然比图灵和邱大不少。简单说，他鼓舞大家去将**证明过程纯机械化**，这样**机器就可以通过形式语言推理出大量定理**（是不是有点像人工智能，机器自己把定理枚举了）。当然那个时代没有今天计算能力如此强大的机器，但当时的科学家们已经在思考今天的事情了。图灵和邱奇都受到这股思潮的影响，几乎从不同的角度解决了同一个问题。无论是图灵机还是λ演算，都可以模拟出我们今天的所有程序。

λ演算是一种**形式系统**(formal system)，什么是形式系统呢？大家知道，**数学语言是是可以脱离现实而存在**的——大家把数学想成了一种符号游戏，脱离生活常识**，从公理开始，进行大量的推导和证明——最终产生了一个系统，里面有公里、定理、推论、猜想...这种自成体系，有公理又承认推理证明方法的体系，称为形式系统**。那么什么是形式语言呢？形式系统需要语言去描绘，这种语言就是形式语言(formal language)。

举一个关于lambda演算的栗子，把我们常见的函数，通过lambda演算计算结果

在数学和计算机科学中，“可计算的”函数是基础观念。对于所谓的可计算性，λ-演算提供了一个简单明确的语义，使计算的性质可以被形式化研究。λ-演算结合了两种简化方式，使得这个语义变得简单。第一种简化是不给予函数一个确定名称，而“匿名”地对待它们。也就是上面的这种形式。（匿名化）

下面举一个普通数学函数，函数化和柯里化的过程。我对lambda也就理解成这样。

square_sum(x,y)=$x^2$+$y^2$   ---匿名的写法--->    (x,y)->$x^2$+$y^2$  （理解成一包含x和y的数组被映射到$x^2$+$y^2$)

同理

id(x)=x   ---匿名的写法--->   x->x（即输入是直接对应到它本身。）

对应到代码就是

```java
 List filters = list.stream().parallel().filter((x) -> x % 2 == 0).collect(Collectors.toList());
```

第二个简化是λ演算只使用单一个参数输入的函数。如果普通函数需要两个参数，例如square\_sum函数，可转成接受单一参数，传给另一个函数中介，而中介函数也只接受一个参数，最后输出结果。（柯里化）

(x,y)->$x^2+y^2$  可以重新写成 x->(y->$x^2+y^2$)  

将多参数的函数转换成为多个中介函数的复合链，每个中介函数都只接受一个参数。这样看有点乱，我们在举一个例子算5的平方加上2的平方。

x->(y->$x^2+y^2$)  (5,2)

=((x->(y->$x^2+y^2$))(5))(2)

=(y->$5^2$+$y^2$)(2)

=$5^2$+$2^2$

=29

在JDK中一般都是匿名化，而没有柯里化。

#### Java中如何实现函数式编程

##### java语言实现函数的参数化

上面说道，函数式编程中函数是一等公民，也就是说函数可以像参数一样(变量一样)传入其他函数中。但是java比较啰嗦

1、java中的方法是不可以独立存在的，必须在类或者接口中定义，这就使得java比较啰嗦，当我们仅仅需要定义一个方法时必须先定义一个类。

2、java中不存在函数指针，所以函数无法被传递。

在JavaScript中可以这样的写代码

```javascript
dealFuction(val,function(a){
	return a>8&&a<100;
});
```

这实现了把行为参数化， 判断 这个行为函数，被当做一个参数传递进入另一个函数，而且判断行为可以在使用时再具体传入，甚至可以临时定义一个传递进去，这样大量的减少了代码量，语言的表达能力大大提高。

不过在java中也可以这样写，不过是通过接口声明的方式。

```java
   new Thread(new Runnable() {
            @Override
            public void run() {
                
            }
        });
```

相信上面的代码，大家都不不会陌生，这是通过接口声明一个方法。然后通过一个匿名内部类去实现，里面的方法。实现了函数的参数化传递。但是这样的写法是不是每次很麻烦，所以java就讲这种东西变成了一个语法糖，也可以说lambda表达式算是一个匿名内部类的简单写法（这样说也不太对，后面会说）。所以JDK8中声明了很多函数式接口，`@FunctionalInterface`。使用这个注解你也可以自定义一些函数方法。然后以参数的方式去传递这个方法。所以感觉JDK为了实现Lamda表达式还是很麻烦；所以JDK也默认给了43个常用的函数式接口，来满足java的API，所以我们基本上自己不用去声明。常用的有`Consumer<T>;Function<T,R>;Predicate<T>;Suppiler<T>`这中接口这里面都会有特定的方法。

在JDK8中引入了函数式的接口。所以一般能

##### 具体分析java中lambda的实现

匿名内部类也是一个类只不过是，没有名字的类，但是JVM还是会个这个内部类命名，他也会存在一个class文件，但是Lambda表达式是没有这个类（想想也是，如果每个lambda表达式都是一个内部类，那如果1000行都是lambda，那类也太多了，所以肯定要有优化）。

```java
public class LamadaMainClient {

    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("hellp");
            }
        });
    }

}
```

![](/img/func/function-02.png)

上面红色的class文件就是，这文件，他产生了匿名内部类`LamadaMainClient$1.class`，而那个绿线是lambda表达式的测试类，没有产生新的类.（会在后面贴出来绿线的java代码）。

Lambda表达式通过invokedynamic指令实现，书写Lambda表达式不会产生新的类。如果有如下代码，编译之后只有一个class文件：

```java
public class MainLambda {
	public static void main(String[] args) {
		new Thread(
				() -> System.out.println("Lambda Thread run()")).start();;
	}
}
```

通过javap反编译命名，我们更能看出Lambda表达式内部表示的不同：

```java
// javap -c -p MainLambda.class
public class MainLambda {
  ...
  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class java/lang/Thread
       3: dup
       4: invokedynamic #3,  0              // InvokeDynamic #0:run:()Ljava/lang/Runnable; /*使用invokedynamic指令调用*/
       9: invokespecial #4                  // Method java/lang/Thread."<init>":(Ljava/lang/Runnable;)V
      12: invokevirtual #5                  // Method java/lang/Thread.start:()V
      15: return

  private static void lambda$main$0();  /*Lambda表达式被封装成主类的私有方法*/
    Code:
       0: getstatic     #6                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #7                  // String Lambda Thread run()
       5: invokevirtual #8                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: return
}
```

既然Lambda表达式不是内部类的简写，那么Lambda内部的this引用也就跟内部类对象没什么关系了。在Lambda表达式中this的意义跟在表达式外部完全一样。因此下列代码将输出两遍Hello Hoolee，而不是两个引用地址。

```java
public class LamadaMainClient {
    
    Runnable r1 = () -> { System.out.println(this); };
    Runnable r2 = () -> { System.out.println(toString()); };
    public static void main(String[] args) {
        new LamadaMainClient().r1.run();
        new LamadaMainClient().r2.run();
    }
    public String toString() { return "Hello lambda"; }
}
//结果是打印两边Hello lambda ，因为这里this，只的是LamadaMainClient这个实例，这个对象会打印Hello lambda。在另一个线程再次，调用所以在次打印Hello lambda 
```

#### StreamAPI的介绍和使用

##### 介绍stream

stream是java函数编程里面的主角，stream并不是某种数据结构，它只是数据源的一种视图。这里的数据源可以是一个数组，Java容器或I/O channel等。（类似与数据库里面的视图）在java中一般用于处理数据，所以一般在集合类中都会有stream的身影，而这个stream也不是我们自己new出来，而是通过集合类中的方法创建出来。

```java
Collection.stream()
Collection.parallelStream()
Arrays.stream(T[] array)
```

##### stream的特性

- 无存储，stream不是一种数据结构，而是数据的视图，是把数据源里的数据映射到stream上。数据源可以是数组，集合。
- 对stream的任何修改都不会修改背后的数据源，比如对stream执行过滤操作并不会删除被过滤的元素，而是会产生一个不包含被过滤元素的新stream。所以一般使用stream的时候都会新生成一个集合或者数组` List<Integer> sortedList = list.stream().sorted(Integer::compareTo).collect(Collectors.toList());`将一个List转换成另外一个List。
- 惰式执行。stream上的操作并不会立即执行，只有等到用户真正需要结果的时候才会执行。（这个会在pipeline，解释）
- 可消费性。stream只能被“消费”一次，一旦遍历过就会失效，就像容器的迭代器那样，想要再次遍历必须重新生成。

##### stream中的操作

- 中间操作：中间操作总是会惰式执行，执行完函数（操作）生成一个新的stream（把一个视图映射到另外一个视图）。
- 结束操作：结束操作会触发实际的计算，计算发生时会把所有中间操作积攒的操作以pipeline（流水线）的方式执行，这样可以减少迭代次数。计算完成之后stream就会失效。（也就是把多个中间操作分成多个流水线，操作组装数据）

##### Stream中常用的API

forEach方法 `void forEach(Consumer<? super E> action); //遍历所有元素`

forEach方法 ：遍历stream中的每一个元素。而且forEach是一个结束操作，所以会马上执行。

```java

    public static void main(String[] args) {
        List<Integer> lists=Lists.newArrayList();
        for (int i = 0; i < 10; i++) {
            lists.add(i);
        }
        lists.parallelStream().forEach((s)-> System.out.println("value:"+s)); //这里使用的并行流，不会依次打印数据
    }
```

filter方法`Stream<T> filter(Predicate<? super T> predicate) //过滤器，过滤符合条件的元素`

filter方法：过滤符合条件的元素，是一个中间操作，不会立即执行。它返回的是一个符合predicate函数的Stream对象。

```java
 // 保留长度等于3的字符串
Stream<String> stream= Stream.of("I", "love", "you", "too");
stream.filter(str -> str.length()==3).forEach(str -> System.out.println(str));
//承接上面主函数中的代码
lists.parallelStream().filter((s)->s>5).forEach((s)-> System.out.println("value:"+s));
```

![](/img/func/Stream.filter.png)

distinct方法`Stream<T> distinct() //去重`

distinct方法：去掉集合中重复的元素，并且返回一个Stream。中间操作

```java
Stream<String> stream= Stream.of("I", "love", "you", "too", "too");
stream.distinct().forEach(str -> System.out.println(str));
```

![](/img/func/Stream.distinct.png)

sort方法`Stream<T>　sorted(Comparator<? super T> comparator)//集合中元素排序`

sort方法：将集合中的元素按照comparator函数排序，返回一个Stream，中间操作。

```java
//承接上面主函数的代码。把集合的元素，倒排序，打印
lists.stream().sorted(Comparator.comparing(Integer::intValue).reversed()).forEach((s)-> System.out.println("value:"+s));
```

 map方法`<R> Stream<R> map(Function<? super T,? extends R> mapper)//对每个元素做处理，返回一个stream`

map方法：就是对每个元素按照某种操作进行转换，转换前后Stream中元素的个数不会改变，但元素的类型取决于转换之后的类型；中间操作。

```java
Stream<String> stream　= Stream.of("I", "love", "you", "too");
stream.map(str -> str.toUpperCase()).forEach(str -> System.out.println(str));
```

![](/img/func/Stream.map.png)

flatMap方法`<R> Stream<R> flatMap(Function<? super T,? extends Stream<? extends R>> mapper)` 作用是对每个元素执行`mapper`指定的操作，并用所有mapper返回的Stream中的元素组成一个新的Stream作为最终返回结果。

flatMap方法:相当于把原Stream中的所有元素都"摊平"之后组成的Stream，转换前后元素的个数和类型都可能会改变。

```java
Stream<List<Integer>> stream = Stream.of(Arrays.asList(1,2), Arrays.asList(3, 4, 5));
stream.flatMap(list -> list.stream()).forEach(i -> System.out.println(i));
```

![](/img/func/Stream.flatMap.png)

collect方法和reduce方法（可以看出来reduction是reduce 的名称）

什么是归约(reduction)：在可计算性理论与计算复杂性理论中，所谓的归约是将某个计算问题转换为另一个问题的过程。

举个栗子：就是乘法化平方的这个过程。

$(a+b)^2$=$a^2$+$b^2$+2ab

上面这个公式我想大家都很熟悉，这是一个非常简单公式。我们根据这个公式把a*b（乘法问题）转化成平方，减法和除法的问题。具体如下面的公式。

$a*b$=$\frac{(a+b)^2-a^2-b^2}{2}$   如果b等于a 那么也会出现$a^2$=$a*a$

也可以是说**给定自然数的集合A和B，是否有可能有效地将用于确定B中的成员转换为用于确定A中的成员的方法。这个过程叫归约**。下面的图片是a*b的集合和$a^2$的集合转换关系（也就是a乘b和a平法集合的一个关系）。也可以说是归纳

![](/img/func/reduction.jpg)

归约操作（reduction operation）又被称作折叠操作（fold），是通过某个连接动作将所有元素汇总成一个汇总结果的过程。元素求和、求最大值或最小值、求出元素总个数、将所有元素转换成一个列表或集合，都属于归约操作。Stream类库有两个通用的规约操作reduce()和collect()，也有一些为简化书写而设计的专用归约操作，比如sum()、max()、min()、count()等。

##### stream中的Pipelines（流水线）

```java
int longestStringLengthStartingWithA
        = strings.stream().filter(s -> s.startsWith("A")).mapToInt(String::length).max();
```

上面的代码是求集合中，A开头的字符串最长的长度。上面这段代码是循环3次，还是循环1次。

![](/img/func/Stream_pipeline_naive.png)

上面图片是通过三次循环处理，第一步是把字符串集合`stream<String>`—>(是否A开头)`List<String>`—>(String转换string的长度)`List<Integer>`—>(求长度值的最大值)`int`。

如果这样的操作的话，我感觉java就有点low，虽然这样操作很简单很明了，知道每个循环中的操作。但是效率不是很高。

首先是迭代次数多。迭代次数跟函数调用的次数相等，而且频繁产生中间结果。每次函数调用都产生一次中间结果，存储开销无法接受。所以Stream中是不会采用这中方法。如果我们自己实现是这样的

```java
int longest = 0;
for(String str : strings){
    if(str.startsWith("A")){// 1. filter(), 保留以A开头的字符串
        int len = str.length();// 2. mapToInt(), 转换成长度
        longest = Math.max(len, longest);// 3. max(), 保留最长的长度
    }
}
return longest;
```

Stream肯定也是上面的写法，但是stream中肯定是要很灵活的。不会每都生成一段代码，所以stream使用的是（pipeline）流水线的方式。添加中间操作就是在流水线添加一个模块，而结束操作就是把中间的处理都做好，然后按动一下按钮，流水线开始工作，集合每一个元素经过中间操作都会去做操作，最后在结束操作那个地方等待被聚合还是被生成其他集合等等。所以stream把用户的每一步操作都抽象成一个中间操作，然后通过流水线来提高效率。（毕竟开发者不知道用户具体什么操作，要他们自己去组装）  

之前我们提到过中间操作和结束操作，说一个放在中间操作（对数据没有操作），等待一个结束操作的时候去触发，然后一起执行。举个例子说也就是像上面代码注释1,2都是中间操作，3取集合的最大值，然后返回给调用方。这里的3属于reduce操作，也可以是说是多一归约。

| 操作                            | 状态                     | 函数                                       |
| ----------------------------- | ---------------------- | ---------------------------------------- |
| 中间操作(Intermediate operations) | 无状态(Stateless)         | unordered() filter() map() mapToInt() mapToLong() mapToDouble() flatMap() flatMapToInt() flatMapToLong() flatMapToDouble() peek() |
|                               | 有状态(Stateful)          | distinct() sorted() sorted() limit() skip() |
| 结束操作(Terminal operations)     | 非短路操作                  | forEach() forEachOrdered() toArray() reduce() collect() max() min() count() |
|                               | 短路操作(short-circuiting) | anyMatch() allMatch() noneMatch() findFirst() findAny() |

Stream上的所有操作分为两类：中间操作和结束操作，中间操作只是一种标记，只有结束操作才会触发实际计算。中间操作又可以分为无状态的(*Stateless*)和有状态的(*Stateful*)，无状态中间操作是指元素的处理不受前面元素的影响，而有状态的中间操作必须等到所有元素处理之后才知道最终结果，比如排序是有状态操作，在读取所有元素之前并不能确定排序结果；结束操作又可以分为短路操作和非短路操作，短路操作是指不用处理全部元素就可以返回结果，比如找到第一个满足条件的元素。之所以要进行如此精细的划分，是因为底层对每一种情况的处理方式不同。

##### stream流水线的实现的简单介绍

我们大致能够想到，应该采用某种方式记录用户每一步的操作，当用户调用结束操作时将之前记录的操作叠加到一起在一次迭代中全部执行掉。沿着这个思路，有几个问题需要解决：

###### 1、用户的操作如何记录？

这里使用的是**操作(operation)**一词，指的是“Stream中间操作”的操作，很多Stream操作会需要一个回调函数（Lambda表达式），因此一个完整的操作是<*数据来源，操作，回调函数*>构成的三元组。（数据来源一般就是collection中数据。操作就是map，filter，collect，forEach等等。回调函数就是map的操作函数或者filter等判定的函数，也就是处理逻辑）

Stream中使用Stage的概念来描述一个完整的操作，并用某种实例化后的PipelineHelper来代表Stage，将具有先后顺序的各个Stage连到一起，就构成了整个流水线。跟Stream相关类和接口的继承关系图示。

![](/img/func/stream_pipeline_classes.png)

图中*Head*用于表示第一个Stage，即调用调用诸如*Collection.stream()方法产生的Stage，很显然这个Stage里不包含任何操作；StatelessOp和StatefulOp*分别表示无状态和有状态的Stage，对应于无状态和有状态的中间操作。

ReferencePipeline类是处理引用类型的流水线，他实现了关于引用类型的操作实现。比如map，filter，forEach等函数的实现。也就是上面提到的都有

```java
//   ReferencePipeline 的内部类，用于引用类型的流水线，从这个类可以看出来，它有一个流(上面的流)，输入类型，操作类型，这也是个枚举（DISTINCT  SORTED  ORDERED  SIZED  SHORT_CIRCUIT）
//这个类主要记录无状态的操作。
abstract static class StatelessOp<E_IN, E_OUT>
            extends ReferencePipeline<E_IN, E_OUT> {
        /**
         * Construct a new Stream by appending a stateless intermediate
         * operation to an existing stream.
         *
         * @param upstream The upstream pipeline stage
         * @param inputShape The stream shape for the upstream pipeline stage
         * @param opFlags Operation flags for the new stage
         */
        StatelessOp(AbstractPipeline<?, E_IN, ?> upstream,
                    StreamShape inputShape,
                    int opFlags) {
            super(upstream, opFlags);
            assert upstream.getOutputShape() == inputShape;
        }

        @Override
        final boolean opIsStateful() {
            return false;
        }
    }
//用于存储有状态的的操作。
    abstract static class StatefulOp<E_IN, E_OUT>
            extends ReferencePipeline<E_IN, E_OUT> {
        /**
         * Construct a new Stream by appending a stateful intermediate operation
         * to an existing stream.
         * @param upstream The upstream pipeline stage
         * @param inputShape The stream shape for the upstream pipeline stage
         * @param opFlags Operation flags for the new stage
         */
        StatefulOp(AbstractPipeline<?, E_IN, ?> upstream,
                   StreamShape inputShape,
                   int opFlags) {
            super(upstream, opFlags);
            assert upstream.getOutputShape() == inputShape; //这里做一个判断，否则抛出异常
        }

        @Override
        final boolean opIsStateful() {
            return true;
        }
		//因为是有状态还要对并行流做处理
        @Override
        abstract <P_IN> Node<E_OUT> opEvaluateParallel(PipelineHelper<E_OUT> helper,
                                                       Spliterator<P_IN> spliterator,
                                                       IntFunction<E_OUT[]> generator);
    }
//引用类型的记录操作的构造函数。他的具体还是要看抽象类，不过能看出来，他只需要上面的流和操作类型
  ReferencePipeline(AbstractPipeline<?, P_IN, ?> upstream, int opFlags) {
        super(upstream, opFlags);
    }
//头部一般用来存储数据，不涉及操作。从构造的参数可以看出来需要一个迭代器，操作类型，和是否并行，然后根据这些条件构建出一个Stream（）
    static class Head<E_IN, E_OUT> extends ReferencePipeline<E_IN, E_OUT> {
        /**
         * Constructor for the source stage of a Stream.
         *
         * @param source {@code Supplier<Spliterator>} describing the stream
         *               source
         * @param sourceFlags the source flags for the stream source, described
         *                    in {@link StreamOpFlag}
         */
        Head(Supplier<? extends Spliterator<?>> source,
             int sourceFlags, boolean parallel) {
            super(source, sourceFlags, parallel);
        }

        /**
         * Constructor for the source stage of a Stream.
         *
         * @param source {@code Spliterator} describing the stream source
         * @param sourceFlags the source flags for the stream source, described
         *                    in {@link StreamOpFlag}
         */
        Head(Spliterator<?> source,
             int sourceFlags, boolean parallel) {
            super(source, sourceFlags, parallel);
        }

        @Override
        final boolean opIsStateful() {
            throw new UnsupportedOperationException();
        }

        @Override
        final Sink<E_IN> opWrapSink(int flags, Sink<E_OUT> sink) {
            throw new UnsupportedOperationException();
        }

        // Optimized sequential terminal operations for the head of the pipeline

        @Override
        public void forEach(Consumer<? super E_OUT> action) {
            if (!isParallel()) {
                sourceStageSpliterator().forEachRemaining(action);
            }
            else {
                super.forEach(action);
            }
        }

        @Override
        public void forEachOrdered(Consumer<? super E_OUT> action) {
            if (!isParallel()) {
                sourceStageSpliterator().forEachRemaining(action);
            }
            else {
                super.forEachOrdered(action);
            }
        }
    }


```

这个是流水线的抽象类的一个构造函数（还有很多构造函数）,所以我们肯定他的数据结构就是一个链表。（所以程序还是算法+数据结构）这门功课还是很重要的。

```java
//从这里面我们可以看出来这个类应该是一个链表的构造函数。而且能看出来他针对是一个记录状态的操作Op的构造函数。
//这个对象存储了向前和向后的引用（指针）。opFlags（操作类型，什么操作）
AbstractPipeline(AbstractPipeline<?, E_IN, ?> previousStage, int opFlags) {
        if (previousStage.linkedOrConsumed)
            throw new IllegalStateException(MSG_STREAM_LINKED);
        previousStage.linkedOrConsumed = true;
        previousStage.nextStage = this;

        this.previousStage = previousStage;
        this.sourceOrOpFlags = opFlags & StreamOpFlag.OP_MASK;
        this.combinedFlags = StreamOpFlag.combineOpFlags(opFlags, previousStage.combinedFlags);
        this.sourceStage = previousStage.sourceStage;
        if (opIsStateful())
            sourceStage.sourceAnyStateful = true;
        this.depth = previousStage.depth + 1;
    }
```

流的输入类型（一般在reference的helper中就是reference，在Int和long就分别对应相对的类型）

```java
enum StreamShape {
    /**
     * The shape specialization corresponding to {@code Stream} and elements
     * that are object references.
     */
    REFERENCE,
    /**
     * The shape specialization corresponding to {@code IntStream} and elements
     * that are {@code int} values.
     */
    INT_VALUE,
    /**
     * The shape specialization corresponding to {@code LongStream} and elements
     * that are {@code long} values.
     */
    LONG_VALUE,
    /**
     * The shape specialization corresponding to {@code DoubleStream} and
     * elements that are {@code double} values.
     */
    DOUBLE_VALUE
}
```

还有IntPipeline, LongPipeline, DoublePipeline没在图中画出，这三个类专门为三种基本类型（不是包装类型）而定制的，跟ReferencePipeline是并列关系。图中Head用于表示第一个Stage，即调用调用诸如Collection.stream()方法产生的Stage，很显然这个Stage里不包含任何操作；**StatelessOp和StatefulOp分别表示无状态和有状态的Stage，对应于无状态和有状态的中间操作**

stream流水线的示意图

![](/img/func/Stream_pipeline_example.png)

图中通过Collection.stream()方法得到Head也就是stage0，紧接着调用一系列的中间操作，不断产生新的Stream。这些Stream对象以双向链表的形式组织在一起，构成整个流水线，由于每个Stage都记录了前一个Stage和本次的操作以及回调函数，依靠这种结构就能建立起对数据源的所有操作。这就是Stream记录操作的方式。

###### 2、操作如何叠加？

以上只是解决了操作记录的问题，要想让流水线起到应有的作用我们需要一种将所有操作叠加到一起的方案。你可能会觉得这很简单，只需要从流水线的head开始依次执行每一步的操作（包括回调函数）就行了。这听起来似乎是可行的，但是你忽略了前面的Stage并不知道后面Stage到底执行了哪种操作，以及回调函数是哪种形式。换句话说，**只有当前Stage本身才知道该如何执行自己包含的动作。这就需要有某种协议来协调相邻Stage之间的调用关系。**

这种协议由*Sink*接口完成，*Sink*接口包含的方法如下表所示：

| 方法名                             | 作用                                       |
| ------------------------------- | ---------------------------------------- |
| void begin(long size)           | 开始遍历元素之前调用该方法，通知Sink做好准备。                |
| void end()                      | 所有元素遍历完成之后调用，通知Sink没有更多的元素了。             |
| boolean cancellationRequested() | 是否可以结束操作，可以让短路操作尽早结束。                    |
| void accept(T t)                | 遍历元素时调用，接受一个待处理元素，并对元素进行处理。Stage把自己包含的操作和回调方法封装到该方法里，前一个Stage只需要调用当前Stage.accept(T t)方法就行了。 |

有了上面的协议，相邻Stage之间调用就很方便了，每个Stage都会将自己的操作封装到一个Sink里，前一个Stage只需调用后一个Stage的`accept()`方法即可，并不需要知道其内部是如何处理的。当然对于有状态的操作，Sink的`begin()`和`end()`方法也是必须实现的。比如Stream.sorted()是一个有状态的中间操作，其对应的Sink.begin()方法可能创建一个乘放结果的容器，而accept()方法负责将元素添加到该容器，最后end()负责对容器进行排序。对于短路操作，`Sink.cancellationRequested()`也是必须实现的，比如Stream.findFirst()是短路操作，只要找到一个元素，cancellationRequested()就应该返回*true*，以便调用者尽快结束查找。Sink的四个接口方法常常相互协作，共同完成计算任务。**实际上Stream API内部实现的的本质，就是如何重载Sink的这四个接口方法**。

有了Sink对操作的包装，Stage之间的调用问题就解决了，执行时只需要从流水线的head开始对数据源依次调用每个Stage对应的Sink.{begin(), accept(), cancellationRequested(), end()}方法就可以了。一种可能的Sink.accept()方法流程是这样的：

```java
void accept(U u){
    //1. 使用当前Sink包装的回调函数处理u
    //2. 将处理结果传递给流水线下游的Sink
}
```

Sink接口的其他几个方法也是按照这种[处理->转发]的模型实现。下面我们结合具体例子看看Stream的中间操作是如何将自身的操作包装成Sink以及Sink是如何将处理结果转发给下一个Sink的。

先看Stream.map()方法：(ReferencePipeline类中的map的实现)

```java
// Stream.map()，调用该方法将产生一个新的Stream
public final <R> Stream<R> map(Function<? super P_OUT, ? extends R> mapper) {
    ...
    return new StatelessOp<P_OUT, R>(this, StreamShape.REFERENCE,
                                 StreamOpFlag.NOT_SORTED | StreamOpFlag.NOT_DISTINCT) {
        @Override /*opWripSink()方法返回由回调函数包装而成Sink*/
        Sink<P_OUT> opWrapSink(int flags, Sink<R> downstream) {
            return new Sink.ChainedReference<P_OUT, R>(downstream) {
                @Override
                public void accept(P_OUT u) {
                    R r = mapper.apply(u);// 1. 使用当前Sink包装的回调函数mapper处理u
                    downstream.accept(r);// 2. 将处理结果传递给流水线下游的Sink
                }
            };
        }
    };
}
```

上述代码看似复杂，其实逻辑很简单，就是将回调函数*mapper*包装到一个Sink当中。由于Stream.map()是一个无状态的中间操作，所以map()方法返回了一个StatelessOp内部类对象（一个新的Stream），调用这个新Stream的opWripSink()方法将得到一个包装了当前回调函数的Sink。

再来看一个复杂一点的例子。Stream.sorted()方法将对Stream中的元素进行排序，显然这是一个有状态的中间操作，因为读取所有元素之前是没法得到最终顺序的。抛开模板代码直接进入问题本质，sorted()方法是如何将操作封装成Sink的呢？sorted()一种可能封装的Sink代码如下：（SortedOps类中关于引用类型操作的排序，内部类的实现）

```java
// Stream.sort()方法用到的Sink实现
class RefSortingSink<T> extends AbstractRefSortingSink<T> {
    private ArrayList<T> list;// 存放用于排序的元素
    RefSortingSink(Sink<? super T> downstream, Comparator<? super T> comparator) {
        super(downstream, comparator);
    }
    @Override
    public void begin(long size) {
        ...
        // 创建一个存放排序元素的列表
        list = (size >= 0) ? new ArrayList<T>((int) size) : new ArrayList<T>();
    }
    @Override
    public void end() {
        list.sort(comparator);// 只有元素全部接收之后才能开始排序
        downstream.begin(list.size());
        if (!cancellationWasRequested) {// 下游Sink不包含短路操作
            list.forEach(downstream::accept);// 2. 将处理结果传递给流水线下游的Sink
        }
        else {// 下游Sink包含短路操作
            for (T t : list) {// 每次都调用cancellationRequested()询问是否可以结束处理。
                if (downstream.cancellationRequested()) break;
                downstream.accept(t);// 2. 将处理结果传递给流水线下游的Sink
            }
        }
        downstream.end();
        list = null;
    }
    @Override
    public void accept(T t) {
        list.add(t);// 1. 使用当前Sink包装动作处理t，只是简单的将元素添加到中间列表当中
    }
}
```

上述代码完美的展现了Sink的四个接口方法是如何协同工作的：

1. 首先beging()方法告诉Sink参与排序的元素个数，方便确定中间结果容器的的大小；
2. 之后通过accept()方法将元素添加到中间结果当中，最终执行时调用者会不断调用该方法，直到遍历所有元素；
3. 最后end()方法告诉Sink所有元素遍历完毕，启动排序步骤，排序完成后将结果传递给下游的Sink；
4. 如果下游的Sink是短路操作，将结果传递给下游时不断询问下游cancellationRequested()是否可以结束处理。

###### 3、叠加之后的操作如何执行？

Sink完美封装了Stream每一步操作，并给出了[处理->转发]的模式来叠加操作。这一连串的齿轮已经咬合，就差最后一步拨动齿轮启动执行。是什么启动这一连串的操作呢？也许你已经想到了启动的原始动力就是结束操作(Terminal Operation)，一旦调用某个结束操作，就会触发整个流水线的执行。

结束操作之后不能再有别的操作，所以结束操作不会创建新的流水线阶段(Stage)，直观的说就是流水线的链表不会在往后延伸了。结束操作会创建一个包装了自己操作的Sink，这也是流水线中最后一个Sink，这个Sink只需要处理数据而不需要将结果传递给下游的Sink（因为没有下游）。对于Sink的[处理->转发]模型，结束操作的Sink就是调用链的出口。

![](/img/func/Stream_pipeline_Sink.png)

我们再来考察一下上游的Sink是如何找到下游Sink的。一种可选的方案是在*PipelineHelper*中设置一个Sink字段，在流水线中找到下游Stage并访问Sink字段即可。但Stream类库的设计者没有这么做，而是设置了一个`Sink AbstractPipeline.opWrapSink(int flags, Sink downstream)`方法来得到Sink，该方法的作用是返回一个新的包含了当前Stage代表的操作以及能够将结果传递给downstream的Sink对象。为什么要产生一个新对象而不是返回一个Sink字段？这是因为使用opWrapSink()可以将当前操作与下游Sink（上文中的downstream参数）结合成新Sink。试想只要从流水线的最后一个Stage开始，不断调用上一个Stage的opWrapSink()方法直到最开始（不包括stage0，因为stage0代表数据源，不包含操作），就可以得到一个代表了流水线上所有操作的Sink，用代码表示就是这样：

```java
// AbstractPipeline.wrapSink()
// 从下游向上游不断包装Sink。如果最初传入的sink代表结束操作，
// 函数返回时就可以得到一个代表了流水线上所有操作的Sink。
final <P_IN> Sink<P_IN> wrapSink(Sink<E_OUT> sink) {
    ...
    for (AbstractPipeline p=AbstractPipeline.this; p.depth > 0; p=p.previousStage) {
        sink = p.opWrapSink(p.previousStage.combinedFlags, sink);
    }
    return (Sink<P_IN>) sink;
}
```

现在流水线上从开始到结束的所有的操作都被包装到了一个Sink里，执行这个Sink就相当于执行整个流水线，执行Sink的代码如下：

```java
    @Override
    final <P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {
        Objects.requireNonNull(wrappedSink);

        if (!StreamOpFlag.SHORT_CIRCUIT.isKnown(getStreamAndOpFlags())) {
            wrappedSink.begin(spliterator.getExactSizeIfKnown()); //通知开始遍历
            spliterator.forEachRemaining(wrappedSink);//遍历
            wrappedSink.end();//通知遍历结束
        }
        else {
            copyIntoWithCancel(wrappedSink, spliterator);
        }
    }


    @Override
    @SuppressWarnings("unchecked")
    final <P_IN> void copyIntoWithCancel(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {
        @SuppressWarnings({"rawtypes","unchecked"})
        AbstractPipeline p = AbstractPipeline.this;
        while (p.depth > 0) {
            p = p.previousStage;
        }
        wrappedSink.begin(spliterator.getExactSizeIfKnown());//通知开始遍历
        p.forEachWithCancel(spliterator, wrappedSink);//遍历
        wrappedSink.end();//通知遍历结束
    }
```

copyInto方法，就是整个流水线处理的一个过程。（上面的三步，也可是说是一个经典的段子。把大象放进冰箱总共分几步）。第一步把冰箱门打开（通知开始是遍历，一些对状态有要求的操作初始化），第二步把大象塞进冰箱（遍历，真正的执行数据操作），第三步把冰箱门带上。（通知遍历结束，不会进入下一个Sink，结束操作）

![](/img/func/copyInto.jpg)

注：图中蓝色线表示数据实际的处理流程

每一个 Sink 都有自己的职责，但具体表现各有不同。无状态操作的 Sink 接收到通知或者数据，处理完了会马上通知自己的 下游。有状态操作的 Sink 则像有一个缓冲区一样，它会等要处理的数据处理完了才开始通知下游，并将自己处理的结果传递给下游。 

例如 sorted() 就是一个有状态的操作，一般会有一个属于自己的容器，用来记录处自己理过的数据的状态。sorted() 是在执行 begin 的时候初始化这个容器，在执行 accept 的时候把数据放到容器中，最后在执行 end 方法时才正在开始排序。排序之后再将数据，采用同样的方式依次传递给下游节点。 

最后数据流到终止节点，终止节点将数据收集起来就结束了。 

然后就没有然后了，copyInto() 返回类型是 void ，没有返回值。 

wrapAndCopyInto() 返回了 TerminalOps 创建的 Sink，这时候它里面已经包含了最终处理的结果。调用它的 get() 方法就获得了最终的结果。 

###### 4、执行后的结果（如果有）在哪里？

首先不是所有的函数式操作都会有返回结果，你如forEach就没有。同时lambda也会有自己的归约操作，没必要使用forEach去处理好多东西。所以forEach一般多多用于打印参数。

```java
// 错误的收集方式
ArrayList<String> results = new ArrayList<>();
stream.filter(s -> pattern.matcher(s).matches())
      .forEach(s -> results.add(s));  // Unnecessary use of side-effects!
// 正确的收集方式
List<String>results =
     stream.filter(s -> pattern.matcher(s).matches())
             .collect(Collectors.toList());  // No side-effects!
```

各种有返回结果的Stream结束操作。

| 返回类型     | 对应的结束操作                           |
| -------- | --------------------------------- |
| boolean  | anyMatch() allMatch() noneMatch() |
| Optional | findFirst() findAny()             |
| 归约结果     | reduce() collect()                |
| 数组       | toArray()                         |

1. 对于表中返回boolean或者Optional的操作（Optional是存放 一个 值的容器）的操作，由于值返回一个值，只需要在对应的Sink中记录这个值，等到执行结束时返回就可以了。
2. 对于归约操作，最终结果放在用户调用时指定的容器中（容器类型通过收集器指定）。collect(), reduce(), max(), min()都是归约操作，虽然max()和min()也是返回一个Optional，但事实上底层是通过调用reduce()方法实现的。
3. 对于返回是数组的情况，毫无疑问的结果会放在数组当中。这么说当然是对的，但在最终返回数组之前，结果其实是存储在一种叫做*Node*的数据结构中的。Node是一种多叉树结构，元素存储在树的叶子当中，并且一个叶子节点可以存放多个元素。这样做是为了并行执行方便。关于Node的具体结构，我们会在下一节探究Stream如何并行执行时给出详细说明。

###### 小结

Stream的设计很巧妙，通过流水线的方式，防止了一个集合操作多次循环问题，这样可以方便我们去写好多东西，同时也提升了效率。但是stream的串性流的的效率还是没有for循环的高，所以在一些对排序没有影响的集合中还是比较推荐使用并行流来去操作数据，可以提高程序的性能，利用多核多线程。



##### java8中增加的函数式编程接口

| 接口定义                  | 接口使用和解释                                  |
| --------------------- | ---------------------------------------- |
| `Consumer<T>`         | 接收一个输入参数，不返回任何结果。T表示输入参数类型（类似消费者）        |
| `Function<T,R>`       | 接收一个输入参数，返回一个结果。T表示输入参数类型，R表示返回结果类型      |
| `Predicate<T>`        | 接受一个输入参数，返回一个布尔值结果 。T表示输入参数类型。返回结果类型BoolBean |
| `Supplier<T>`         | 不接受任何参数，返回一个结果。T是结果的类型。（类似与生产者）          |
| `BiConsumer<T,U>`     | 代表了一个接受两个输入参数的操作，并且不返回任何结果。T，U两种参数类型     |
| `BiFunction<T,U,R>`   | 代表了一个接受两个输入参数的方法，并且返回一个结果。T，U代表输入参数类型。R代表输出类型 |
| `UnaryOperator<T>`    | 接受一个参数为类型T,返回值类型也为T。这个和Function有点类似。     |
| `ToLongFunction<T>`   | 接受一个输入T类型参数，返回一个long类型结果。不一定是泛型          |
| `IntBinaryOperator`   | 接受两个参数同为类型int,返回值类型也为int。（也是Function类似）  |
| `IntToDoubleFunction` | 接受一个int类型输入，返回一个double类型结果 。             |
| `IntPredicate`        | 接受一个int输入参数，返回一个布尔值的结果。                  |
| `DoubleUnaryOperator` | 接受一个参数同为类型double,返回值类型也为double 。         |
| ` IntSupplier`        | 无参数，返回一个int类型结果。                         |

尽管上面介绍了这么多函数式接口，JDK还有好多函数式编程的接口我这边没有列出来，而且本文也不会一一拿出来去解释，只是挑几种主要的形式来简单说一下。其实你看到这个列表估计也能猜出来一些。

`Predicate<T> `用于判别一个对象。一般返回一个Boolean对象。这类函数里面的主要的方法是`boolean test(T t);`。比如求一个人是否为男性

`Consumer<T>`用于接收一个对象进行处理但没有返回，这类函数里主要方法是`void accept(T t);`比如接收一个人并打印他的名字

`Function<T,R>`转换一个对象为不同类型的对象。这类函数里主要方法是`R apply(T t);`比如方形转换成圆形。

`Supplier<T>`不接收任何参数，返回一个对象T。这类函数里主要方法是`T get();`生成唯一Id值。

BinaryOperator(T, T)T接收两个同类型的对象，并返回一个原类型对象

##### 常用的一些函数式编程和stream使用案例

```java

public class LamadaTestClient {

    public String outFunction(Integer x) {
        System.out.println("this Function Integer:" + x);
        return "this Function Integer:" + x;
    }

    public void outConsumer(String s) {
        System.out.println("this Conumer String:" + s);
    }

    public Boolean outPredicate(Integer i) {
        System.out.println("i:" + i + (i > 0));
        return i > 0;
    }

    public Integer outSuppiler() {
        System.out.println("this need return value:" + 1000);
        return 1000;
    }

    public void outBiConsumer(Integer x, String str) {
        System.out.println("param:" + x + "string" + str);
    }

    public void outOK(Integer i) {
        System.out.println("say OK" + i);
    }

    //接收一个参数返回一个的 函数对象
    @Test
    public void testFunction() {
        Function<Integer, String> function = (x) -> outFunction(x);
        String string = function.apply(1000);
    }

    @Test
    public void testConsumer() { //消费者
        Consumer<String> consumer = (s) -> outConsumer(s);
        consumer.accept("what is that");
    }

    @Test
    public void testPredicate() {  //返回一个boolean值，根据T
        Predicate<Integer> predicate = (i) -> outPredicate(i);
        predicate.test(10);
    }

    @Test
    public void testSupplier() { //生产者
        Supplier<Integer> supplier = () -> outSuppiler();
        supplier.get();
    }

    @Test
    public void testBiConsumer() {
        BiConsumer<Integer, String> consumer = (x, str) -> outBiConsumer(x, str);
        consumer.accept(10, "wulili");

    }

    @Test
    public void testOK() {
        Consumer<Integer> consumer = LamadaTestClient.this::outOK; //函数引用
        consumer.accept(1000);
    }

    @Test
    public void testMaxAndMin() {
        List<Integer> list = Lists.newArrayList(3, 5, 2, 9, 1);
        int maxInt = list.stream().max(Integer::compareTo).get();
        System.out.println(maxInt);
        int minInt = list.stream().min(Integer::compareTo).get();
        System.out.println(minInt);
    }

  	//stream中常用的方法。
    @Test
    public void testReduce() {
        int sum = Stream.of(1, 2, 3, 4, 5, 6, 7).reduce(0, (a, b) -> a + b);
        System.out.println("reduce 求和" + sum);
        int sum1 = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9).reduce(1, (a, b) -> a * b);
        System.out.println("reduce 乘积" + sum1);
        int pSum = Stream.of(1, 2, 3, 4, 5, 6, 7).parallel().reduce(0, (a, b) -> a + b);
        System.out.println("reduce 并行求和" + pSum);
        int pSum1 = Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9).parallel().reduce(1, (a, b) -> a * b);
        System.out.println("reduce 并行求和" + pSum1);
        int sumSize = Stream.of("Apple", "Banana", "Orange", "Pear").parallel().map(s -> s.length()).reduce(Integer::sum).get();
        System.out.println("mapReduce求和" + sumSize);
    }

    @Test
    public void testSort() {
        //从小到大排序
        List<Integer> list = Lists.newArrayList(3, 5, 1, 10, 8);
        List<Integer> sortedList = list.stream().sorted(Integer::compareTo).collect(Collectors.toList());
        System.out.println(sortedList);
        //从大到小排序
        List<Integer> sortedList1 = list.stream().sorted(Comparator.comparing(Integer::intValue).reversed()).collect(Collectors.toList());
        System.out.println(sortedList1);
        //从大到小排序,使用并行操作实现排序
        List<Integer> sortedListP = list.stream().parallel().sorted(Comparator.comparing(Integer::intValue).reversed()).collect(Collectors.toList());
        System.out.println(sortedListP);

    }

    @Test
    public void testStreamDistinct() {
        List<String> list = Lists.newArrayList("a", "b", "c", "d", "a", "d");
        List dis = list.stream().parallel().distinct().collect(Collectors.toList());
        System.out.println(dis);
    }

    @Test
    public void testStreamFilter() {
        List<Integer> list = IntStream.range(0, 100).boxed().collect(Collectors.toList());
        List filters = list.stream().parallel().filter((x) -> x % 2 == 0).collect(Collectors.toList());
        System.out.println(filters);

    }

    @Test
    public void testStreamMap() {
        List<String> list = Lists.newArrayList("d", "a", "e", "a", "b", "c");
        List maps = list.stream().parallel().distinct().sorted(String::compareTo).map((x) -> x.hashCode()).collect(Collectors.toList());
        System.out.println(maps);
    }

    @Test
    public void testStreamFlatmap() {
        String poetry = "Where, before me, are the ages that have gone?\n" +
                "And where, behind me, are the coming generations?\n" +
                "I think of heaven and earth, without limit, without end,\n" +
                "And I am all alone and my tears fall down.";
        Stream<String> lines = Arrays.stream(poetry.split("\n"));
        List<String> arr = Arrays.stream(poetry.split("\n")).collect(Collectors.toList());
        System.out.println(arr);
        Stream<String> words = lines.flatMap(line -> Arrays.stream(line.split(" ")));
        List<String> arrw = Arrays.stream(poetry.split("\n")).flatMap(line -> Arrays.stream(line.split(" "))).collect(Collectors.toList());
        System.out.println(arrw);
        List<String> flatList = words.map(w -> {
            if (w.endsWith(",") || w.endsWith(".") || w.endsWith("?"))
                return w.substring(0, w.length() - 1).trim().toLowerCase();
            else
                return w.trim().toLowerCase();
        }).distinct().sorted().collect(Collectors.toList());
        System.out.println(flatList);

    }

    @Test
    public void testSteamLimit() {
        List<Integer> list = IntStream.range(0, 100).filter((x) -> x % 2 == 0).boxed().limit(20).collect(Collectors.toList());
        System.out.println(list);
    }

    @Test
    public void testSteamPeek() {
        IntStream.range(0, 100).filter((x) -> x % 2 == 1).boxed().limit(20).peek((x) -> System.out.println("hello world num:" + x)).collect(Collectors.toList());
    }

    @Test
    public void testSteamSkip() {
        List<Integer> list = IntStream.range(0, 100).skip(50).filter((x) -> x % 2 == 0).boxed().peek((x -> System.out.println("nice num:" + x))).sorted(Comparator.comparing(Integer::intValue).reversed()).collect(Collectors.toList());
        System.out.println(list);
    }

    @Test
    public void testSteamMatch(){
        System.out.println(Stream.of(1,2,3,4,5,6).allMatch((x)->x>0));//所有的匹配
        System.out.println(Stream.of(-1,2,3,4,5,6).allMatch((x)->x>0));
        System.out.println(Stream.of(-1,-2,-3,-4,-5).anyMatch((x)->x>0));//只要有一个匹配
        System.out.println(Stream.of(-1,-2,-3,-4,5).anyMatch((x)->x>0));
        System.out.println(Stream.of(-1,-2,-3,-4,5).noneMatch((x)->x>0));//不要任何匹配
        System.out.println(Stream.of(-1,-2,-3,-4,-5).noneMatch((x)->x>0));
    }
    @Test
    public void testStreamCount(){
        System.out.println( IntStream.range(0,100).filter((x)->x%2==0).boxed().count());
    }

    @Test
    public void testOptional() {
        System.out.println(Optional.empty());
    }
}
```

#### 总结

##### java引进函数式编程的优缺点

1、代码简洁函数式编程写出的代码简洁且意图明确，极大的提高编程效率和程序可读性，使用stream接口让你从此告别for循环。

2、多核友好，Java函数式编程使得编写并行程序从未如此简单，你需要的全部就是调用一下parallel()方法。

3、对集合对象进行各种非常便利、高效的聚合操作（aggregate operation），或者大批量数据操作 (bulk data operation)。

4、同时它提供串行和并行两种模式进行汇聚操作，并发模式能够充分利用多核处理器的优势，使用 fork/join 并行方式来拆分任务和加速处理过程。

5、可能对于代码断点调试不是特别友好，不能像之前的可以在for循环里面单步的执行。

6、在没有使用并行流的话，在性能上可能可能不如之前的for循环。（随着数据量的增增大）。

在平时开发的过程中如果喜欢使用这种函数式的写法，就可以大胆的去实践，而且现在大多数公司也都普及JDK8（其实JDK8 都有点过时的，java的下一个稳定版本已经出来了，以后java里面也可以有类型推断）。单但是更重要的是代码的可读性和逻辑性（其实最重要的是正确性）。

#### 杂记

JDK8中大概有43个，函数式接口[相关43个接口和简单的解释](http://www.runoob.com/java/java8-functional-interfaces.html)，我这里面好多都是借鉴上面的。

参考大佬的github   [JavaLambdaInternals](https://github.com/CarpenterLee/JavaLambdaInternals) 

lambda演算的解释   [认知科学家写给小白的Lambda演算](https://zhuanlan.zhihu.com/p/30510749)

参考的博客 https://paper.tuisec.win/detail/827ee2ddea083d0

柯里化：在lambda演算中，每个函数都只能有一个参数，当多个函数参数传递的时候，将多参数的函数转换成为多个中介函数的复合链，每个中介函数都只接受一个参数。

```javascript
function add(a, b) { return a + b; };
var add = function (a) { return function (b) {return a + b;};};
```

Spliterator（可拆分迭代器）

我们知道`Iterator`是用来迭代容器的，`Spliterator`也有类似作用，但二者有如下不同：

1. `Spliterator`既可以像`Iterator`那样逐个迭代，也可以批量迭代。批量迭代可以降低迭代的开销。
2. `Spliterator`是可拆分的，一个`Spliterator`可以通过调用`Spliterator<T> trySplit()`方法来尝试分成两个。一个是`this`，另一个是新返回的那个，这两个迭代器代表的元素没有重叠。

可通过（多次）调用`Spliterator.trySplit()`方法来分解负载，以便多线程处理。

操作StreamOpFlag的枚举，这里面枚举用的很巧妙，使用二进制来表示一些东西。他是type枚举的组合。这里是枚举的部分代码，里面还有很多，但是这些就基本可以知道大概的一些东西。01表示设置或者注入值，10表示清空值，11表示保留值。DISTINCT=0,SORTED=1,ORDERED=2,SIZED=3,SHORT_CIRCUIT=12;他们在使用的过程中可以使用或|， 与&这种逻辑运算。

操作StreamOpFlag的枚举，这里面枚举用的很巧妙，使用二进制来表示一些东西。他是type枚举的组合。这里是枚举的部分代码，里面还有很多，但是这些就基本可以知道大概的一些东西。01表示设置或者注入值，10表示清空值，11表示保留值。DISTINCT=0,SORTED=1,ORDERED=2,SIZED=3,SHORT_CIRCUIT=12;他们在使用的过程中可以使用或|， 与&这种逻辑运算。

```java
enum StreamOpFlag {

    /*
     * Each characteristic takes up 2 bits in a bit set to accommodate
     * preserving, clearing and setting/injecting information.
     *
     * This applies to stream flags, intermediate/terminal operation flags, and
     * combined stream and operation flags. Even though the former only requires
     * 1 bit of information per characteristic, is it more efficient when
     * combining flags to align set and inject bits.
     *
     * Characteristics belong to certain types, see the Type enum. Bit masks for
     * the types are constructed as per the following table:
     *
     *                        DISTINCT  SORTED  ORDERED  SIZED  SHORT_CIRCUIT
     *          SPLITERATOR      01       01       01      01        00
     *               STREAM      01       01       01      01        00
     *                   OP      11       11       11      10        01
     *          TERMINAL_OP      00       00       10      00        01
     * UPSTREAM_TERMINAL_OP      00       00       10      00        00
     *
     * 01 = set/inject
     * 10 = clear
     * 11 = preserve
     *
     * Construction of the columns is performed using a simple builder for
     * non-zero values.
     */


    // The following flags correspond to characteristics on Spliterator
    // and the values MUST be equal.
    //

    /**
     * Characteristic value signifying that, for each pair of
     * encountered elements in a stream {@code x, y}, {@code !x.equals(y)}.
     * <p>
     * A stream may have this value or an intermediate operation can preserve,
     * clear or inject this value.
     */
    // 0, 0x00000001
    // Matches Spliterator.DISTINCT
    DISTINCT(0,
             set(Type.SPLITERATOR).set(Type.STREAM).setAndClear(Type.OP)),

    /**
     * Characteristic value signifying that encounter order follows a natural
     * sort order of comparable elements.
     * <p>
     * A stream can have this value or an intermediate operation can preserve,
     * clear or inject this value.
     * <p>
     * Note: The {@link java.util.Spliterator#SORTED} characteristic can define
     * a sort order with an associated non-null comparator.  Augmenting flag
     * state with addition properties such that those properties can be passed
     * to operations requires some disruptive changes for a singular use-case.
     * Furthermore, comparing comparators for equality beyond that of identity
     * is likely to be unreliable.  Therefore the {@code SORTED} characteristic
     * for a defined non-natural sort order is not mapped internally to the
     * {@code SORTED} flag.
     */
    // 1, 0x00000004
    // Matches Spliterator.SORTED
    SORTED(1,
           set(Type.SPLITERATOR).set(Type.STREAM).setAndClear(Type.OP)),

    /**
     * Characteristic value signifying that an encounter order is
     * defined for stream elements.
     * <p>
     * A stream can have this value, an intermediate operation can preserve,
     * clear or inject this value, or a terminal operation can preserve or clear
     * this value.
     */
    // 2, 0x00000010
    // Matches Spliterator.ORDERED
    ORDERED(2,
            set(Type.SPLITERATOR).set(Type.STREAM).setAndClear(Type.OP).clear(Type.TERMINAL_OP)
                    .clear(Type.UPSTREAM_TERMINAL_OP)),

    /**
     * Characteristic value signifying that size of the stream
     * is of a known finite size that is equal to the known finite
     * size of the source spliterator input to the first stream
     * in the pipeline.
     * <p>
     * A stream can have this value or an intermediate operation can preserve or
     * clear this value.
     */
    // 3, 0x00000040
    // Matches Spliterator.SIZED
    SIZED(3,
          set(Type.SPLITERATOR).set(Type.STREAM).clear(Type.OP)),

    // The following Spliterator characteristics are not currently used but a
    // gap in the bit set is deliberately retained to enable corresponding
    // stream flags if//when required without modification to other flag values.
    //
    // 4, 0x00000100 NONNULL(4, ...
    // 5, 0x00000400 IMMUTABLE(5, ...
    // 6, 0x00001000 CONCURRENT(6, ...
    // 7, 0x00004000 SUBSIZED(7, ...

    // The following 4 flags are currently undefined and a free for any further
    // spliterator characteristics.
    //
    //  8, 0x00010000
    //  9, 0x00040000
    // 10, 0x00100000
    // 11, 0x00400000

    // The following flags are specific to streams and operations
    //

    /**
     * Characteristic value signifying that an operation may short-circuit the
     * stream.
     * <p>
     * An intermediate operation can preserve or inject this value,
     * or a terminal operation can preserve or inject this value.
     */
    // 12, 0x01000000
    SHORT_CIRCUIT(12,
                  set(Type.OP).set(Type.TERMINAL_OP));

    // The following 2 flags are currently undefined and a free for any further
    // stream flags if/when required
    //
    // 13, 0x04000000
    // 14, 0x10000000
    // 15, 0x40000000

    /**
     * Type of a flag
     */
    enum Type {
        /**
         * The flag is associated with spliterator characteristics.
         */
        SPLITERATOR,

        /**
         * The flag is associated with stream flags.
         */
        STREAM,

        /**
         * The flag is associated with intermediate operation flags.
         */
        OP,

        /**
         * The flag is associated with terminal operation flags.
         */
        TERMINAL_OP,

        /**
         * The flag is associated with terminal operation flags that are
         * propagated upstream across the last stateful operation boundary
         */
        UPSTREAM_TERMINAL_OP
    }
}
```

