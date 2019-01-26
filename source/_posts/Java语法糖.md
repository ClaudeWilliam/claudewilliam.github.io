---
title: Java语法糖
date: 2017-05-14 11:26:05
tags: 编程
categories: java
---
- Java语法糖（syntactic sugar），也称为糖衣语法，是由英国计算机科学家Peter.j.Landin发明的术语，指计算机语言中添加某种语法。（说白了就是对现有语法的封装）这种语法对语言是我功能并没有影响，但是方便程序员使用。Java中最常用的语法糖泛型，变长参数，条件编译，自动拆装箱，内部类，枚举类等。虚拟机其实并不支持这些语法，他们都是在编译期被还原成简单基础的语法结构。这个过程为语法糖。
- 泛型的实现
```java
/**
* 在源代码中存在泛型
*/
public static void main(String[] args) {
    Map<String,String> map = new HashMap<String,String>();
    map.put("hello","你好");
    String hello = map.get("hello");
    System.out.println(hello);
}

//当上述源代码被编译为class文件后，泛型被擦除且引入强制类型转换
public static void main(String[] args) {
    HashMap map = new HashMap(); //类型擦除
    map.put("hello", "你好");
    String hello = (String)map.get("hello");//强制转换
    System.out.println(hello);
}
```
- 自动拆装箱的实现
```
//源码中的泛型
public static void main(String[] args) {
    Integer a = 1;
    int b = 2;
    int c = a + b;
    System.out.println(c);
}

//编译为class文件

public static void main(String[] args) {
    Integer a = Integer.valueOf(1); // 自动装箱
    byte b = 2;
    int c = a.intValue() + b;//自动拆箱
    System.out.println(c);
}
```
- 变长参数
```
//源码中的变长参数
public class Varargs {
    public static void print(String... args) {
        for(String str : args){
            System.out.println(str);
        }
    }

    public static void main(String[] args) {
        print("hello", "world");
    }
}
//编译后的变长参数，而且能看出来变长参数是通过数组实现的
public class Varargs {
    public Varargs() {
    }

    public static void print(String... args) {
        String[] var1 = args;
        int var2 = args.length;
        //增强for循环的数组实现方式
        for(int var3 = 0; var3 < var2; ++var3) {
            String str = var1[var3];
            System.out.println(str);
        }

    }

    public static void main(String[] args) {
        //变长参数转换为数组
        print(new String[]{"hello", "world"});
    }
}


```
- 内部类
```
//在源码中的内部类
public class Outer {
    class Inner{
    }
}

//在编译后的内部类

class Outer$Inner {
    Outer$Inner(Outer var1) {
        this.this$0 = var1;
    }
}

```
- 枚举类型
```
//在源码中的枚举实现

public enum Fruit {
    APPLE,ORINGE
}

//编译后的枚举

//继承java.lang.Enum并声明为final
public final class Fruit extends Enum
{

    public static Fruit[] values()
    {
        return (Fruit[])$VALUES.clone();
    }

    public static Fruit valueOf(String s)
    {
        return (Fruit)Enum.valueOf(Fruit, s);
    }

    private Fruit(String s, int i)
    {
        super(s, i);
    }
    //枚举类型常量
    public static final Fruit APPLE;
    public static final Fruit ORANGE;
    private static final Fruit $VALUES[];//使用数组进行维护

    static
    {   
        //protected Enum(String name, int ordinal),这个构造函数是Enum自带的，ordinal是用来排序的
        APPLE = new Fruit("APPLE", 0);
        ORANGE = new Fruit("ORANGE", 1);
        $VALUES = (new Fruit[] {
            APPLE, ORANGE
        });
    }
}
```