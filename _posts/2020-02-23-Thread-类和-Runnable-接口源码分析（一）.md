---
layout:     post
title:      Thread 类和 Runnable 接口源码分析（一）
subtitle:   Java 并发学习系列博客
date:       2020-02-23
author:     GJXAIOU 
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Java 并发
---

##  Thread 类与 Runnable 接口 JavaDoc 分析

### （一）两者关系

首先 Thread 类是实现了 Runnable 接口 `public class Thread implements Runnable {`

### （二）Thread 类分析

**下面为完整的 Thread 类的 JavaDoc 文档**

> 一个 Thread 是程序的一个执行线程，Java虚拟机允许应用程序同时运行多个执行线程。
>
> 每个线程都有一个优先级，优先级较高的线程优先于优先级较低的线程执行。每个线程也可以标记为守护进程，也可以不标记为守护进程。当在某个线程中运行的代码创建一个新的 Thread 对象时，新线程的优先级最初设置为等于创建线程的优先级（即如果在一个线程中创建另一个线程，则被创建线程初始优先级和创建它的线程优先级相同），并且仅当创建线程是守护进程时被创建的线程才是守护进程线程。
>
> 当Java虚拟机启动时，通常只有一个非守护进程线程（它通常调用某些指定类的名为 main 的方法，所以 main 方法是执行在线程上的）。Java虚拟机会继续执行线程，直到发生以下任一情况：
>
> - 类 Runtime 的 exit 方法被调用，并且类安全管理器允许退出操作发送；
>
>     - 不是守护进程线程的所有线程都已死亡，消亡原因可能是调用 run 方法返回了，或者抛出了超过 run 方法范围的异常。
>
> 有两种方法可以创建一个新的执行线程，一种是将类声明为 Thread 类的子类。这个子类应该覆盖重写Thread 类的 run 方法。然后可以分配并启动子类的实例。例如，计算大于指定值的素数的线程可以如下编写：
>
> ```java
> class PrimeThread extends Thread {
>    long minPrime;
>    PrimeThread(long minPrime) {
>        this.minPrime = minPrime;
>    }
>    public void run() {
>        // compute primes larger than minPrime
>        ...
>    }
> }
> ```
>
> 然后通过如下代码可以创建一个线程然后开始运行
>
> ```java
> PrimeThread p = new PrimeThread(143);
> p.start();
> ```
>
> 创建线程的另一种方法是声明一个实现  Runnable 接口的类。然后，该类实现 run 方法。然后可以分配类的实例（即可以创建该类的实例），在创建 Thread 时将该实例对象作为参数传递，然后启动。其他样式中的相同示例如下所示：
>
> ```java
> class PrimeRun implements Runnable {
>     long minPrime;
>     PrimeRun(long minPrime) {
>         this.minPrime = minPrime;
>     }
> 
>     public void run() {
>         // compute primes larger than minPrime
>         ......
>     }
> }
> ```
>
> 然后创建线程对象，启动
>
> ```java
> PrimeRun p = new PrimeRun(143);
> new Thread(p).start();
> ```
>
> 每个线程都有一个用于标识的名称。多个线程可能具有相同的名称。如果在创建线程时未指定名称，则会为其生成新名称。

---

**在 Thread 类的代码中同样含有设置了线程优先级代码，代码如下：**

```java
/**
* The minimum priority that a thread can have.
*/
public final static int MIN_PRIORITY = 1;

/**
* The default priority that is assigned to a thread.
*/
public final static int NORM_PRIORITY = 5;

/**
* The maximum priority that a thread can have.
*/
public final static int MAX_PRIORITY = 10;
```

注意构造方法中的参数含义，start 方法、run 方法。



### （三）Runnable 接口（函数型接口）

**下面为 Runnable 接口的 JavaDoc 文档**

> `Runnable` 接口应该由其实例要由线程执行的任何类实现。类必须定义一个没有参数的 run 方法。
>
> 此接口旨在为希望在活动时执行代码的对象提供通用协议。例如，Thread 类实现了 Runnable 接口。
>
> **处于活动状态只意味着线程已启动但尚未停止**。
>
> 此外，`Runnable` 提供了一种方法，使类在不子类化 `Thread` 类的情况下处于活动状态。实现 Runnable 接口的类可以通过实例化 Thread 实例并将其自身作为目标传入而运行，而无需子类化 Thread 类。在大多数情况下，如果您只计划重写 run（）方法，而不打算重写其他 Thread 类的方法，则应使用  Runnable 接口。这一点很重要，因为除非程序员打算修改或增强类的基本行为，否则不应该对类进行子类化（即不应该继承）。

```java
package java.lang;

/*
 * @author  Arthur van Hoff
 * @see     java.lang.Thread
 * @see     java.util.concurrent.Callable
 * @since   JDK1.0
 */
@FunctionalInterface
public interface Runnable {
    /**
    // 当实现 Runnable 接口的对象被用于创建一个线程的时候，启动该线程的时候会导致该对象的 run 方法在独立执行的线程中被调用。
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}

```



