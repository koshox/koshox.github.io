---
title: 一种独特的单例模式写法，利用final语义实现
date: 2020-03-23 13:08:11
tags:
 - Java
 - JVM
 - Design pattern
categories:
 - Java
---

单例模式作为Java老生常谈的东西，大家再熟悉不过，基本是"茴字的四种写法"问题，有静态内部类，枚举，DCL等等。

最近看见一种比较新奇的写法 — 利用final语义实现线程安全的单例模式。

<!-- more -->

## final语义

final除了除了修饰类，方法，属性表示不可变以外，还有一个很重要的语义，即读写重排序规则：

1. 在构造函数内对一个 final 域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
2. 初次读一个包含 final 域的对象的引用，与随后初次读这个 final 域，这两个操作之间不能重排序。

为什么会有这样的一个规则呢：

> 在旧的Java内存模型中，一个最严重的缺陷就是线程可能看到final域的值会改变。比如，一个线程当前看到一个整型final域的值为0（还未初始化之前的默认值），过一段时间之后这个线程再去读这个final域的值时，却发现值变为1（被某个线程初始化之后的值）。最常见的例子就是在旧的Java内存模型中，String的值可能会改变。
>
> 为了修补这个漏洞，JSR-133专家组增强了final的语义。通过为final域增加写和读重排序规则，可以为Java程序员提供初始化安全保证：只要对象是正确构造的（被构造对象的引用在构造函数中没有“逸出”），那么不需要使用同步（指lock和volatile的使用）就可以保证任意线程都能看到这个final域在构造函数中被初始化之后的值。
>
> —《Java并发编程的艺术》

#### final域的写规则

- JMM 禁止编译器把 final 域的写重排序到构造函数之外。
- 编译器会在 final 域的写之后，构造函数 return 之前，插入一个 StoreStore 屏障。这个屏障禁止处理器把 final 域的写重排序到构造函数之外。
- 在构造函数内对一个 final 引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

final域的写规则可以保证：在对象引用为任意线程可用之前，对象的final域已经被正确初始化了，而普通域不具备这个保障，即普通域在实际构造函数中初始化时可能会被处理器重排序到构造函数之外。

#### final域的读规则

在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这两个操作（这个规则仅仅针对处理器）。编译器的实现是在读final域的前面插入一个 LoadLoad屏障。

final域的读规则可以保证：在读一个对象的 final 域之前，一定会先读包含这个 final 域的对象的引用，如果该引用不为 null，那么引用对象的 final 域一定已经被 A 线程初始化过了。

## DCL单例模式

先看一下比较常见的双重检查锁写法，比较重要的是其中的volatile关键字。

```java
public class Singleton {
    private volatile static Singleton INSTANCE;

    private Singleton() {}

    public static Singleton getSingleton() {
        if (INSTANCE == null) {
            synchronized (Singleton.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}
```

Java创建一个对象有如下的步骤：

1. 申请内存空间
2. 初始化默认值
3. 执行构造器方法
4. 连接引用和实例

其中3和4有可能会重排序，造成判断 INSTANCE == null为false的时候，对象可能尚未初始化完成。volatile可以用来禁止指令重排序，来解决这个问题。

## final实现单例模式

从DCL单例已经可以发现，volatile是为了防止指令重排序，既然final语义确保了写不会重排序到构造函数之外，同样可以保证可见性，是不是用final也可以呢？的确可以。

这种写法是在CMU的SEI CERT wiki上看到的：https://wiki.sei.cmu.edu/confluence/display/java/LCK10-J.+Use+a+correct+form+of+the+double-checked+locking+idiom

本质上是用final代替volatile的DCL。

```java
public final class Helper {
  private final int n;
  
  public Helper(int n) {
    this.n = n;
  }
  
  // Other fields and methods, all fields are final
}
  
final class Foo {
  private Helper helper = null;
  
  public Helper getHelper() {
    Helper h = helper;       // Only unsynchronized read of helper
    if (h == null) {
      synchronized (this) {
        h = helper;          // In synchronized block, so this is safe
        if (h == null) {
          h = new Helper(42);
          helper = h;
        }
      }
    }
    return h;
  }
}
```

值得一提的是，SEI CERT特意举了个不完全正确的例子：

```java
// 不正确
final class Foo {
  private Helper helper = null;
  
  public Helper getHelper() {
    if (helper == null) {            // First read of helper
      synchronized (this) {
        if (helper == null) {        // Second read of helper
          helper = new Helper(42);
        }
      }
    }
    return helper;                   // Third read of helper
  }
}
```

> However, this code is not guaranteed to succeed on all Java Virtual Machine platforms because there is no [happens-before relationship](https://wiki.sei.cmu.edu/confluence/display/java/Rule+BB.+Glossary#RuleBB.Glossary-happens-beforeorder) between the first read and third read of `helper`. Consequently, it is possible for the third read of `helper` to obtain a stale null value (perhaps because its value was cached or reordered by the compiler), causing the `getHelper()` method to return a null pointer.

大意是这种写法不能在所有的JVM上都成功，因为第一次和第三次的读取没有建立happens-before关系，可能会因为编译器缓存或重排序导致获取到null值。

关于这个说法呢，我大概测试了一下，Jdk1.8 Hotspot上这么写是没毛病的，从final语义和程序次序规则看，第一次读取不为null的话后面也不应该为null才对，可能是作者说的编译器缓存或者不同虚拟机实现不同。

Java happens-before规则：

>1. 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作；
>2. 锁定规则：一个unLock操作先行发生于后面对同一个锁额lock操作；
>3. volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作；
>4. 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C；
>5. 线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作；
>6. 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生；
>7. 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行；
>8. 对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始；

## 最舒服的单例模式-Guava

这篇文章主要是想探讨final语义，来更多的了解Java并发编程，并不是真的推荐在代码里这么用。

我比较喜欢用Guava的Suppliers（本质还是DCL）来实现单例模式，简洁清晰，DCL自己手写比较长，而静态内部类和枚举还需要创建一个class。

```java
public class Singleton {
    private static final Supplier<Singleton> INSTANCE = Suppliers.memoize(Singleton::new);
    
    private Singleton() {}
    
    public static Singleton getSingleton() {
        return INSTANCE.get();
    }
}
```