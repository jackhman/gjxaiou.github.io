---
layout:     post
title:      notify、notifyAll 方法详解以及线程获取锁的方式
subtitle:   Java 并发学习系列博客
date:       2020-02-25
author:     GJXAIOU 
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Java 并发
---



## notify、notifyAll 方法详解以及线程获取锁的方式

### （一） notify 方法分析

下面为 notify 方法的 JavaDoc

---

> 唤醒等待此对象锁的单个线程。如果有任何（多个）线程正在等待这个对象（的锁），则选择其中一个等待的线程被唤醒。选择是任意的，由实现者自行决定。线程通过调用 wait 方法中的一个（因为有多个 wait）来等待对象的监视器。
>
> 在当前线程放弃对该对象的锁定之前，唤醒的线程将无法继续（执行）。唤醒的线程将以通常的方式与任何其他线程竞争，这些线程竞争在此对象上的同步；例如，唤醒的线程在成为下一个锁定此对象的线程时没有可靠的特权或劣势（和其它线程地位相同）。
>
> 此方法只能由作为此对象监视器所有者的线程调用（即调用了 notify 方法的线程一定是持有当前对象锁的线程）。线程通过以下三种方式之一成为对象锁的持有者：
>
> - 通过执行该对象的同步实例方法（即被标记为 Synchronized 的实例方法）。
> - 通过执行在对象上同步的{@code synchronized}语句的主体（即执行 synchronized 的语句块）。
> - 对于 class 类型的对象，通过执行该类的同步静态方法(即被标记为 Synchronized 的静态方法)
>     在某一时刻只有一个线程可以拥有该对象的锁的。

----

```java
/*
 * @throws  IllegalMonitorStateException  if the current thread is not
 *               the owner of this object's monitor.
 * @see        java.lang.Object#notifyAll()
 * @see        java.lang.Object#wait()
 */
public final native void notify();
```

### （二）notifyAll 方法分析

首先是介绍 notifyAll 方法的 JavaDoc

>**唤醒此对象监视器上等待的所有线程**。线程通过调用 wait 方法之一来等待对象的监视器。
>
>在当前线程放弃对该对象的锁定之前，唤醒的线程将无法继续。唤醒的线程将以通常的方式与任何其他线程竞争，这些线程可能正在积极竞争以在此对象上同步；例如，唤醒的线程在成为下一个锁定此对象的线程时没有可靠的特权或劣势。
>
>此方法只能由作为此对象监视器所有者的线程调用。请参阅{@code notify}方法，以了解线程成为监视器所有者的方式。

```java
   /**
     * @throws  IllegalMonitorStateException  if the current thread is not
     *               the owner of this object's monitor.
     * @see        java.lang.Object#notify()
     * @see        java.lang.Object#wait()
     */
public final native void notifyAll();
```



### （三）wait 和 notify 、notifyAll 方法与线程同步系统总结

- 当调用 wait 方法时，首先需要确保**调用的 wait 方法的线程已经持有了对象的锁**；
- 当调用 wait 方法后，该线程就会释放掉这个对象的锁，然后进入到等待状态（或者成为等待集合：wait set)；
- 当线程调用了 wait 之后进入到等待状态时候，它就可以等待其他线程调用相同对象的 notify 或 notifyAll 方法来使得自己被唤醒；
- 一旦这个线程被其他线程唤醒之后，该线程就会与其他线程一同开始竞争这个对象的锁（公平竞争）；**只有当该线程获取到了这个对象的锁之后**，代码才会继续向下执行，没有获取到则继续等待。
- 调用 wait 方法的代码片段需要放在一个 synchronized 块或者是 synchronized 方法中，这样才可以确保线程在调用 wait 方法前已经获取到了对象的锁。
- 当调用对象的 notify 方法时，它会随机唤醒该对象等待集合（wait set) 中的任意一个线程，当某个线程被唤醒之后，他就会与其他线程一同竞争对象的锁。
- 当调用对象的 notifyAll 方法时候，他就会唤醒该对象等待集合（wait set) 中的所有线程，这些线程被唤醒之后，又会开始竞争对象的锁；
- 在某一个时刻，只有唯一一个线程可以拥有对象的锁。



### （四）wait 和 notify 方法案例剖析和详解

编写一个多线程程序，实现目标：

- 存在一个对象，该对象（即对象对应的类中）存在一个 int 类型的成员变量 counter，该成员变量的初始值为 0；
- 创建两个线程，其中一个线程对该对象的成员变量 counter 增1，另一个线程对该对象的成员变量减一；
- 输出该对象成员变量 counter 每次变化后的值；
- 最终输出的结果应该为：1010101010…



**分析**

- 需要两个线程，因为操作不同，所以需要两个线程类；

首先构造 MyObject 类

```java
package com.gjxaiou;

/**
 * @Author GJXAIOU
 * @Date 2020/2/15 10:51
 */
public class MyObject {
    private int counter;

    // 分别实现递增、递减方法，因为要调用 wait 方法，所以需要使用 synchronized
    public synchronized void increase() {
        // 进入 increase 方法之后，如果 counter 不为 0，则该线程等待
        while (counter != 0) {
            try {
                // 调用 wait 使得释放掉 counter 的锁，使得递减的线程拿到锁
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        // 当另一个线程执行完递减之后，调用 notify 方法让递增线程唤醒了
        counter++;
        System.out.println(counter);
        // 通知其它线程起来（这里就是指递减的线程）
        notify();
    }

    // 下面的解释同上
    public synchronized void decrease() {
        while (counter == 0) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        counter--;
        System.out.println(counter);
        notify();
    }
}
```

然后分别实现递增和递减两个线程类

```java
package com.gjxaiou;

/**
 * @Author GJXAIOU
 * @Date 2020/2/15 11:18
 */
public class IncreaseThread extends Thread {
    private MyObject myObject;

    public IncreaseThread(MyObject myObject) {
        this.myObject = myObject;
    }

    // 重写 run 方法

    @Override
    public void run() {
        for (int i = 0; i < 30; i++) {
            // 每次先让线程休眠 0 ~ 1 秒
            try {
                Thread.sleep((long) (Math.random() * 1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            myObject.increase();
        }
    }
}

```

```java
package com.gjxaiou;

/**
 * @Author GJXAIOU
 * @Date 2020/2/15 11:24
 */
public class DecreaseThread extends Thread {
    private MyObject myObject;

    public DecreaseThread(MyObject myObject) {
        this.myObject = myObject;
    }

    // 重写 run 方法
    @Override
    public void run() {
        for (int i = 0; i < 30; i++) {
            // 每次先让线程休眠 0 ~ 1 秒
            try {
                Thread.sleep((long) (Math.random() * 1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            myObject.decrease();
        }
    }
}

```

最后在 main 方法中创建分别创建一个递增和递减线程，然后执行程序

```java
package com.gjxaiou;

/**
 * @Author GJXAIOU
 * @Date 2020/2/15 11:26
 */
public class Client {
    public static void main(String[] args) {
        // 只能创建一个对象，因为锁是加在对象上，如果两个线程操作两个对象就没有意义了MyObject myObject = new MyObject();
        MyObject myObject = new MyObject();
        Thread increaseThread = new IncreaseThread(myObject);
        Thread decreaseThread = new DecreaseThread(myObject);
        increaseThread.start();
        decreaseThread.start();
    }
}
```

程序运行结果：（这里选取前十结果）

```java
1
0
1
0
1
0
1
0
1
0
```

