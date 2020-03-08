---
layout:     post
title:      wait 和 sleep 方法字节码分析
subtitle:   Java 并发学习系列博客
date:       2020-02-24
author:     GJXAIOU 
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Java 并发
---



## wait 和 sleep 方法字节码分析

### （一）Object 类中的关于线程的方法

主要包括 wait（） 、notify 、notifyAll 方法

```java
/*
 * Copyright (c) 1994, 2012, Oracle and/or its affiliates. All rights reserved.
 * ORACLE PROPRIETARY/CONFIDENTIAL. Use is subject to license terms.
 */

package java.lang;

/**
 * Class {@code Object} is the root of the class hierarchy.
 * Every class has {@code Object} as a superclass. All objects,
 * including arrays, implement the methods of this class.
 *
 * @author  unascribed
 * @see     java.lang.Class
 * @since   JDK1.0
 */
public class Object {

    public final native void notify();

    public final native void notifyAll();
   
    public final native void wait(long timeout) throws InterruptedException;
   
    public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos >= 500000 || (nanos != 0 && timeout == 0)) {
            timeout++;
        }

        wait(timeout);
    }
}    
```

### （二）wait 方法详解

#### 1.无参数 wait 方法分析

**wait 方法的 JavaDoc**，这里是针对无参数的 wait() 方法：

> 会使当前线程等待，直到另一个线程调用此对象的 `java.lang.Object.notify()` 方法或`java.lang.Object.notifyAll()` 方法。（因此 wait 和 notify或者 notifyall 方法总是成对出现的，wait 会使当前线程出现等待，直到另一个线程调用了当前这个对象的 notify 或者 notifyAll 方法才会使当前线程被唤醒）
>
> 换句话说，这个方法的行为就像它只是执行调用 wait（0）(因为 wait() 里面的参数为超时时间，如果不写或者是 wait(0) 就是一直等待)。
>
> 当前线程必须拥有此对象的监视器（通常就是指锁）。（当这个线程调用了 wait 方法之后）线程释放此监视器的所有权并等待，直到另一个线程通过调用 `notify` 方法或 `notifyAll` 方法通知等待此对象监视器唤醒的线程。然后线程还需等待，直到它可以重新获得监视器的所有权并继续执行。

上面总结：

  - 如果想调用 wait 方法，当前线程必须拥有这个对象的锁
  - 一旦调用 wait 方法之后，调用该方法的线程就会释放被他所调用的对象的锁，然后进入等待状态。
   - 一直等待到另外的线程去通知在这个对象的锁上面等待的所有的线程（因为有可能一个对象，有多个线程都调用了这个对象的 wait 方法，这样多个线程都会陷入等待的状态）

> 在单参数版本中，中断和虚假唤醒是可能的，此方法应始终在循环中使用：
>
> ```java
> synchronized (obj) {
>  while (<条件尚未满足>)
>      obj.wait();
>   ...//Perform action appropriate to condition
> } 
> ```
>
> **此方法只能由作为此对象监视器所有者的线程调用**。请参阅 notify方法，以了解线程成为监视器所有者的方式（线程如何获取对象的锁见 notify 方法）。

```java
/* @throws  IllegalMonitorStateException  if the current thread is not
     *               the owner of the object's monitor.
     * @throws  InterruptedException if any thread interrupted the
     *             current thread before or while the current thread
     *             was waiting for a notification.  The <i>interrupted
     *             status</i> of the current thread is cleared when
     *             this exception is thrown.
     * @see        java.lang.Object#notify()
     * @see        java.lang.Object#notifyAll()
     */
public final void wait() throws InterruptedException {
    wait(0);
}
```

----

【针对上面的 wait 方法】所以当前线程必须持有对象的锁才能调用 wait 方法，而且一旦调用 wait 方法之后就会将调用了 wait 对象的锁释放掉。同时线程还有一个 sleep 方法（Thread 类中）。



上面分析的是不带参数的 wait 方法，该方法实际实际上是调用了带参数的 wait 方法（见下），传入的是参数 0，接下来分析带参数的 wait 方法

```java
  public final void wait() throws InterruptedException {
        wait(0);
    }
```

#### 2.带参数 wait 方法分析

-----

> 使当前线程等待，直到另一个线程调用此对象的 notify（）方法或 notifyAll（）方法，或者指定的时间已过。
> 当前线程必须拥有此对象的监视器（锁）。
>
> 此方法会导致当前线程（称为 T 线程）将自身放置在此对象的等待集合中，然后放弃此对象上的任何和所有同步声明（即是释放锁）。线程 T 出于线程调度目的被禁用，并处于休眠状态，直到发生以下四种情况之一：
>
> - 其他一些线程调用了此对象的 notify 方法，而线程 T 恰好被任意选择为要唤醒的线程（从等待集合中选择一个）。
> - 其他一些线程为此对象调用 notifyAll 方法。
> - 其他一些线程中断了线程 T 。
> - 指定的时间或多或少已经过去。但是，如果参数 timeout 为零，则不考虑实时性，线程只需等待通知。
>
> 然后从该对象的等待集合删除线程 T （因为已经被唤醒了），并重新启用线程调度。然后，它以通常的方式与其他线程去竞争对象上的同步权；一旦它获得了对象的控制权（可以对这个对象同步了），它对对象的所有同步声明都将恢复到原来的状态，也就是说，恢复到调用 wait 方法时的所处的状态。然后线程 T 从 wait方法的调用中返回。因此，从 wait 方法返回时，对象和线程 T 的同步状态与调用 wait 方法时完全相同。
>
> 线程也可以在不被通知、中断或超时的情况下唤醒，即所谓的“虚假唤醒”。虽然这种情况在实践中很少发生，但应用程序必须通过测试本应导致线程被唤醒的条件，并在条件不满足时继续等待来防范这种情况。换句话说，等待应该总是以循环的形式出现，如下所示
>
> ```java
> // 对 obj 对象同步和上锁
> synchronized (obj) {
>     while (<condition does not hold>)
>     // 当另一个线程调用 obj 的 notify 方法的时候，正好当前线程就是被唤醒的线程的话，就会从这里唤醒然后执行一系列操作，然后再次判断
>        obj.wait(timeout);
>        ... // Perform action appropriate to condition
> }
> ```
>
> 如果当前线程在等待之前或等待期间被任何线程中断，则抛出一个 InterruptedException。在还原此对象的锁定状态（如上所述）之前，不会引发此异常。
>
> 注意，wait 方法在将当前线程放入此对象的等待集合中时，只解锁此对象；在线程等待期间，当前线程可能同步的任何其他对象都将保持锁定状态（因为一个线程在执行的时候可能同时调用几个对象的 wait 方法，但是某个时刻通过 notify 方法唤醒线程之后，但是其他对象还保持锁定）。
>
> 此方法只能由作为此对象监视器所有者的线程调用。查看 notify 方法，了解线程成为监视器所有者的方式

----

```java
/*
     * @param      timeout   the maximum time to wait in milliseconds.// 超时时间，如果为 0 表示一直等待。
     * @throws  IllegalArgumentException      if the value of timeout is
     *               negative.
     * @throws  IllegalMonitorStateException  if the current thread is not
     *               the owner of the object's monitor.
     * @throws  InterruptedException if any thread interrupted the
     *             current thread before or while the current thread
     *             was waiting for a notification.  The <i>interrupted
     *             status</i> of the current thread is cleared when
     *             this exception is thrown.
     * @see        java.lang.Object#notify()
     * @see        java.lang.Object#notifyAll()
     */
public final native void wait(long timeout) throws InterruptedException;
```

另一个 wait 方法和上面 wait 一样，只不多加了一个设置纳秒的参数，可以实现更加精确的时间，可以从下面代码中看出最后还是调用了上面的一个参数的 wait 方法；

```java
// 这里的 JavaDoc 和上面相似，只是多了 nanos 说明以及使用方式
public final void wait(long timeout, int nanos) throws InterruptedException {
    if (timeout < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException(
            "nanosecond timeout value out of range");
    }

    if (nanos >= 500000 || (nanos != 0 && timeout == 0)) {
        timeout++;
    }

    wait(timeout);
}
```



### （三）Thread 类的 sleep 方法详解

从下面Thread 类的 sleep 方法对应的 JavaDoc 可以看出 sleep 会一直持有该对象的锁，不会释放掉。

> 会导致当前正在执行的线程进入休眠状态（临时的停止执行一段特定时间的毫秒数），它会受到系统定时器和调度器的精度的影响。线程并不会失去任何锁的所有权。

```java
/* @param  millis
     *         the length of time to sleep in milliseconds
     *
     * @throws  IllegalArgumentException
     *          if the value of {@code millis} is negative
     *
     * @throws  InterruptedException
     *          if any thread has interrupted the current thread. The
     *          <i>interrupted status</i> of the current thread is
     *          cleared when this exception is thrown.
     */
public static native void sleep(long millis) throws InterruptedException;
```



测试代码为：

```java
package com.gjxaiou;

/**
 * @Author GJXAIOU
 * @Date 2020/2/14 11:23
 */

/**
 * 在调用 wait 方法时，线程必须要持有被调用对象的锁，当调用 wait 方法后，线程就会释放掉该对象的锁
 * 在调用 Thread 类的 sleep 方法时候，线程是不会释放掉对象的锁的。
 */
public class MyTest1 {
    public static void main(String[] args) throws InterruptedException {

        Object object = new Object();
        // 测试：线程必须用于对象的锁才能调用 wait 方法，如果直接调用会报错
        // Exception in thread "main" java.lang.IllegalMonitorStateException
        // 即当前的线程一定要持有调用 wait 对象（这里是 object 对象）的锁才可以
        // 解决方法：可以将调用 wait 方法放入 synchronized 同步代码块，因为进入代码块中就相当于获取到对象的锁了
        // object.wait();
        synchronized (object) {
            // 进入代码块相当于已经获取到 object 对象的锁
            object.wait();
        }
    }
}

```

对编译之后的 MyTest1.class 进行反编译之后得到（下面仅仅为 main 方法中反编译结果）

```java
 public static void main(java.lang.String[]) throws java.lang.InterruptedException;
    Code:
       0: new           #2                  // class java/lang/Object
       3: dup
       4: invokespecial #1                  // Method java/lang/Object."<init>":()V
       7: astore_1
       8: aload_1
       9: dup
      10: astore_2
      11: monitorenter // 注：当执行 synchronized 代码块，一旦进入对应的字节码指令为 monitorenter
      12: aload_1
      13: invokevirtual #3                  // Method java/lang/Object.wait:()V
      16: aload_2
      17: monitorexit // 注：从 synchronized 代码块正常或非正常退出都对应着 monitorexit 指令
      18: goto          26
      21: astore_3
      22: aload_2
      23: monitorexit // 这是异常的退出对应的 monitoexit 指令
      24: aload_3
      25: athrow
      26: return

```



**小结**

- 在调用 wait 方法时候，线程必须要持有被调用对象的锁，当调用 wait 方法之后，线程就会释放掉该对象的锁（monitor）。

- 在调用 Thread 类的 sleep 方法时候，线程是不会释放掉对象的锁的。