---
layout:     post
title:      synchronized 关键字原理详解（字节码、自旋）
subtitle:   Java 并发学习系列博客
date:       2020-02-27
author:     GJXAIOU 
header-img: img/post-bg-debug.png
catalog: true
tags:
    - Java 并发
---



## synchronized 关键字原理详解（字节码、自旋）

### （一）引言

**关于 synchronized 的一道题目**

```java
public class Test{
    public synchronized void method1(){
    }
    
    public synchronized void method2(){
    }
}
// 当创建 Test 的唯一一个实例（对象）
Test test = new Test():
// 当一个线程执行这个对象的 method1 时候（还没有执行完），其他线程能不能执行该对象的 method2

```

【解答】不能，因为对于同一个对象来说，它的所有 synchronize 方法锁的对象是同一个东西，当一个线程正在执行其中的一个 synchronized 方法的时候，其他线程执行不了其他方法的，因为该 synchronized 方法已经被该线程进去了，已经获取该对象的锁了。

如果创建了两个对象就可以了，因为一个线程获取一个对象的锁并不妨碍另一个线程获取另一个对象的锁。

- 代码修改：

    ```java
    public class Test{
        public synchronized void method1(){
        }
        
        public static synchronized void method2(){
        }
    }
    // 当创建 Test 的唯一一个实例（对象）
    Test test = new Test():
    // 当一个线程执行这个对象的 method1 时候（还没有执行完），其他线程能不能执行该对象的 method2
    ```

    可以，因为第一个方法的 synchronized 关键字锁的是当前的  test 对象，而第二个 方法的synchronized 关键字锁的是当前 test 对象所对应的 Class 对象（因为本质上一个静态方法并不属于当前对象，属于当前对象所对应的 Class 对象），是两个独立的对象，两个对立的对象拥有各自独立的锁。

### （二）测试

#### 1.测试程序一：

```java
package com.gjxaiou;

/**
 * @Author GJXAIOU
 * @Date 2020/2/15 15:16
 */
public class MyThreadTest {
    public static void main(String[] args) {
        Runnable thread = new MyThread();
        // 因为创建线程 thread1 和 thread2 时候传入的是同一个 Runnable 实例对象
        Thread thread1 = new Thread(thread);
        Thread thread2 = new Thread(thread);
        // 因此当两个线程启动 start 方法时候都会去执行同一个 MyThread 对象里面的 run 方法
        thread1.start();
        thread2.start();
    }
}

// 该线程类实现了 Runnable 接口
class MyThread implements Runnable {
    int x;

    public void run() {
        x = 0;
        while (true) {
            System.out.println("result:" + x++);

            try {
                Thread.sleep((long) (Math.random() * 1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            if (x == 30) {
                break;
            }
        }
    }
}
```



程序执行结果：

```java
result:0
result:0
result:1
result:2
result:3
result:4
result:5
result:6
result:7
result:8
result:9
result:10
result:11
result:12
result:13
result:14
result:15
result:16
result:17
result:18
result:19
result:20
result:21
result:22
result:23
result:24
result:25
result:26
result:27
result:28
result:29

Process finished with exit code 0

```

通过下面一个小的框架示例来说明上面程序

```java
// struts2
public class LoginAction{
    private String username;
    private String password;
}
```

这两个成员变量分别对应着用户请求表单中对应的两个参数。参数的信息是以成员变量的形式放置到类中的，但是成员变量可能会被多线程修改的。所以 Struts 的 LoginAction 就是一个多实例的，就是用户每一次访问登录都会创建一个新的实例。这样只有一个线程会访问到同一个实例。

```java
public class LoginController{
	public void login(String username, String password){
	}
}
```

在 SpringMVC 中，当用户登录的时候，表单中的信息就会映射到方法的两个参数中，而对于一个方法而言，无论方法的参数还是方法内部代码声明的变量，都是局部变量。而局部变量是归一个线程所独有。所以在 Controller 中一般不会定义可以被修改的成员变量。一般都是放置只读的或者无状态的变量。



#### 2.测试程序二：

```java
package com.gjxaiou;

/**
 * @Author GJXAIOU
 * @Date 2020/2/15 16:00
 */
public class MyThreadTest2 {
    public static void main(String[] args) {
        // 使用此种方式输出为：hello,world
        MyClass myClass = new MyClass();
        Thread1 thread1 = new Thread1(myClass);
        Thread2 thread2 = new Thread2(myClass);

        // 测试方式二：
        // 如果使用以下代码代替上面代码，结果为 ：world,hello
//        MyClass myClass = new MyClass();
//        MyClass myClass1 = new MyClass();
//        Thread1 thread1 = new Thread1(myClass);
//        Thread2 thread2 = new Thread2(myClass1);

        thread1.start();
        // 休眠一段时间
        try {
            Thread.sleep(700);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        thread2.start();
    }
}

class MyClass {
    public synchronized void hello() {
        // thread1 首先进入 hello 方法，即是下面休眠了，但是不会释放对 myClass 对象的锁
        // 所以即是上面主线程在 700 毫秒之后恢复了，接着 thread2 启动，然后访问 world 方法，因为这时候 myClass 对象的锁还在 thread1 中，所以不能访问。
        try {
            Thread.sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("hello");
    }

    public synchronized void world() {
        System.out.println("world");
    }
}

// 定义两个线程类
class Thread1 extends Thread {
    private MyClass myClass;

    public Thread1(MyClass myClass) {
        this.myClass = myClass;
    }

    @Override
    public void run() {
        myClass.hello();
    }
}

class Thread2 extends Thread {
    private MyClass myClass;

    public Thread2(MyClass myClass) {
        
        
        this.myClass = myClass;
    }

    @Override
    public void run() {
        myClass.world();
    }
}
```

如果一个对象中含有若干个 synchronized 方法，那么在某一个时刻只能有唯一的线程进入到其中一个 synchronized 方法。其他线程即使想访问其他 synchronized 方法也要等待。因为当一个线程想要访问其中一个 synchronized 方法的时候，要尝试着获取当前对象的锁（而当前对象只有唯一的一把锁）。



### （三）透过字节码理解 synchronized 关键字

- synchronized 关键字一般用于修饰一个方法或者修饰一个代码块

    - 修饰方法

        方法可以是静态或者非静态的，如果是修饰实例方法（不加 static 关键字），当线程去访问的该方法的时候，是给当前对象上锁。如果是修饰静态方法，线程访问该方法的的时候是给该对象对应的类的 Class 对象上锁。

    - 修饰代码块：

        synchronized 关键字 后面会跟上一个对象的名字（引用的名字），加上具体执行的代码逻辑。



**总结**：当我们使用 synchronized 关键字来修饰代码块时候，字节码层面上是通过 monitorenter 和 monitorexit 指令来实现锁的获取与释放动作。

当线程进入到 monitorenter 指令后，线程将会持有被同步的对象（就是 synchronized 关键值后面括号中的对象）的 monitor 对象，当退出 monitorenter 指令之后（即执行 monitorexit 指令），线程将会释放该 monitor 对象。

#### 1. synchronized 关键字修饰代码块

**测试程序1：**

```java
package com.gjxaiou.synchronize;

/**
 * 测试 synchronized 用法
 *
 * @Author GJXAIOU
 * @Date 2020/2/16 21:51
 */
public class MyTest1 {
    // 同步代码块（因为一个方法中可能只有几行需要上锁，所以关键字即可）
    private Object object = new Object();

    public void method() {
        // 表示对 object 对象上锁，当执行到这里的时候该线程会尝试获取 object 对象的锁，如果获取到就就行执行，如果获取不到就阻塞了。
        synchronized (object) {
            System.out.println("hello world");
        }
    }
}
```

**反编译之后的结果** 更加具体的反编译：包括常量池信息

```java
E:\Program\Java\Project\JavaConcurrency\ProficientInJavaConcurrency\target\classes\com\gjxai
ou\synchronize>javap -v MyTest1.class
Classfile /E:/Program/Java/Project/JavaConcurrency/ProficientInJavaConcurrency/target/classe
s/com/gjxaiou/synchronize/MyTest1.class
  Last modified 2020-2-16; size 624 bytes
  MD5 checksum 1f26b481eb5d16a1179ccfb6138ef19e
  Compiled from "MyTest1.java"
public class com.gjxaiou.synchronize.MyTest1
  minor version: 0
  major version: 49
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   // 常量池省略
{
     // 编译器自动生成的构造方法
  public com.gjxaiou.synchronize.MyTest1();
   // 构造方法省略

  public void method();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1 // 最大栈深度、局部变量表的数量和参数的数量（因为 Java 中任何实例方法的第一个参数都是 this(不是在定义或者声明该方法的时候显式指定的，而是编译器在程序编译完成之后动态传入的)，所以我们可以在方法中使用 this 关键字来引用当前对象的成员变量或者其他方法）
         0: aload_0
         1: getfield      #3                  // Field object:Ljava/lang/Object; // 表示获取当前对象的成员变量（因为 synchronized 要对 object 对象进行同步，所以要先获取该对象） 
         4: dup
         5: astore_1
         6: monitorenter  // 执行完该行助记符之后就进入了同步方法里面了
         7: getstatic     #4   // Field java/lang/System.out:Ljava/io/PrintStream;
             // 首先要获取到 system.out 对象（该对象实际为一个 System 类中的静态变量，点进去看看，类型是 java.lang.PrintStream）
        10: ldc           #5                  // String hello world
        12: invokevirtual #6// Method java/io/PrintStream.println:(Ljava/lang /String;)V  // 调用了 out 对象中的 println 方法
        15: aload_1
        16: monitorexit  // 锁退出(正常退出）
        17: goto          25
        20: astore_2
        21: aload_1
        22: monitorexit  // 出现异常时候退出，保证该线程无论十分情况下都可以释放掉该对象的锁
        23: aload_2
        24: athrow
        25: return
      Exception table:
         from    to  target type
             7    17    20   any
            20    23    20   any
      LineNumberTable:
        line 15: 0
        line 16: 7
        line 17: 15
        line 18: 25
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      26     0  this   Lcom/gjxaiou/synchronize/MyTest1;
}
SourceFile: "MyTest1.java"
```



**测试程序 2**：如果两个方法中都含有 synchronized 修饰的代码块

```java
package com.gjxaiou.synchronize;

/**
 * 测试 synchronized 用法
 *
 * @Author GJXAIOU
 * @Date 2020/2/16 21:51
 */
public class MyTest1 {
    // 同步代码块（因为一个方法中可能只有几行需要上锁，所以关键字即可）
    private Object object = new Object();

    public void method() {
        // 表示对 object 对象上锁，当执行到这里的时候该线程会尝试获取 object 对象的锁，如果获取到就就行执行，如果获取不到就阻塞了。
        synchronized (object) {
            System.out.println("hello world");
        }
    }

    public void method2(){
        synchronized (object){
            System.out.println("welcome");
        }
    }
}

```

反编译之后的结果为：**每个方法都生成了一个 monitorenter 和 2 个 monitorexit**

```java
E:\Program\Java\Project\JavaConcurrency\ProficientInJavaConcurrency\target\classes\c
om\gjxaiou\synchronize>javap -v MyTest1.class
Classfile /E:/Program/Java/Project/JavaConcurrency/ProficientInJavaConcurrency/targe
t/classes/com/gjxaiou/synchronize/MyTest1.class
  Last modified 2020-2-17; size 757 bytes
  MD5 checksum 2908e6ef3bf3551cae84950f5a1bf947
  Compiled from "MyTest1.java"
public class com.gjxaiou.synchronize.MyTest1
  minor version: 0
  major version: 49
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
// 常量池省略
{
  public com.gjxaiou.synchronize.MyTest1();
// 构造方法省略

  public void method();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: getfield      #3                  // Field object:Ljava/lang/Object;
         4: dup
         5: astore_1
         6: monitorenter
         7: getstatic     #4                  // Field java/lang/System.out:Ljava/io
/PrintStream;
        10: ldc           #5                  // String hello world
        12: invokevirtual #6                  // Method java/io/PrintStream.println:
(Ljava/lang/String;)V
        15: aload_1
        16: monitorexit
        17: goto          25
        20: astore_2
        21: aload_1
        22: monitorexit
        23: aload_2
        24: athrow
        25: return
      Exception table:
         from    to  target type
             7    17    20   any
            20    23    20   any
      LineNumberTable:
        line 15: 0
        line 16: 7
        line 17: 15
        line 18: 25
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      26     0  this   Lcom/gjxaiou/synchronize/MyTest1;

  public void method2();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: getfield      #3                  // Field object:Ljava/lang/Object;
         4: dup
         5: astore_1
         6: monitorenter
         7: getstatic     #4                  // Field java/lang/System.out:Ljava/io
/PrintStream;
        10: ldc           #7                  // String welcome
        12: invokevirtual #6                  // Method java/io/PrintStream.println:
(Ljava/lang/String;)V
        15: aload_1
        16: monitorexit
        17: goto          25
        20: astore_2
        21: aload_1
        22: monitorexit
        23: aload_2
        24: athrow
        25: return
      Exception table:
         from    to  target type
             7    17    20   any
            20    23    20   any
      LineNumberTable:
        line 21: 0
        line 22: 7
        line 23: 15
        line 24: 25
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      26     0  this   Lcom/gjxaiou/synchronize/MyTest1;
}
SourceFile: "MyTest1.java"

```



**测试程序 3：**在上面代码中增加了抛出异常，字节码对应几个 monitorenter  和 monitorexit

```java
package com.gjxaiou.synchronize;

/**
 * 测试 synchronized 用法
 *
 * @Author GJXAIOU
 * @Date 2020/2/16 21:51
 */
public class MyTest1 {
    // 同步代码块（因为一个方法中可能只有几行需要上锁，所以关键字即可）
    private Object object = new Object();

    public void method() {
        // 表示对 object 对象上锁，当执行到这里的时候该线程会尝试获取 object 对象的锁，如果获取到就就行执行，如果获取不到就阻塞了。
        synchronized (object) {
            System.out.println("hello world");
            throw new RuntimeException();
        }
    }

    public void method2(){
        synchronized (object){
            System.out.println("welcome"):;
        }
    }
}
 	
```

反编译之后结果为：

**method1 方法对应字节码**为什么只有一个 monitorexit，因为代码块中代码无论是 print 语句还是 throw 语句抛出异常，该代码块最后都是以异常结束的。最终异常抛出对应 27 行的 athrow 助记符（ 22 行的 athrow 对应于显式的 throw new runtimeXX 动作。

**method2 方法对应字节码**中仍然为两个 monitorexit，因为程序有可能正常结束，正常结束的时候程序就从 17 行 goto 到 25 行（跳过了中间异常退出）直接 return，如果要是抛出异常的话，会在想 22 行那样先释放锁，然后在 24 行抛出异常。

```java
E:\Program\Java\Project\JavaConcurrency\ProficientInJavaConcurrency\target\classes\com\gjxaiou\synchronize>javap -v MyTest1.class
Classfile /E:/Program/Java/Project/JavaConcurrency/ProficientInJavaConcurrency/target/classes/com/gjxaiou/synchronize/MyTest1.class
  Last modified 2020-2-17; size 788 bytes
  MD5 checksum b98e0d981195ca3814a94fc4872ecb95
  Compiled from "MyTest1.java"
public class com.gjxaiou.synchronize.MyTest1
  minor version: 0
  major version: 49
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
 // 常量池省略
{
  public com.gjxaiou.synchronize.MyTest1();
   // 构造函数省略
  public void method();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: getfield      #3                  // Field object:Ljava/lang/Object;
         4: dup
         5: astore_1
         6: monitorenter
         7: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        10: ldc           #5                  // String hello world
        12: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        15: new           #7                  // class java/lang/RuntimeException
        18: dup
        19: invokespecial #8  // 因为 new 出来一个 RuntimeException 实例，所以调用其的构造方法                // Method java/lang/RuntimeException."<init>":()V
        22: athrow
        23: astore_2
        24: aload_1
        25: monitorexit // 代码块执行结束释放锁
        26: aload_2
        27: athrow
      Exception table:
         from    to  target type
             7    26    23   any
      LineNumberTable:
        line 15: 0
        line 16: 7
        line 17: 15
        line 18: 23
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      28     0  this   Lcom/gjxaiou/synchronize/MyTest1;

  public void method2();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: getfield      #3                  // Field object:Ljava/lang/Object;
         4: dup
         5: astore_1
         6: monitorenter
         7: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        10: ldc           #9                  // String welcome
        12: invokevirtual #6                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        15: aload_1
        16: monitorexit
        17: goto          25
        20: astore_2
        21: aload_1
        22: monitorexit
        23: aload_2
        24: athrow
        25: return
      Exception table:
         from    to  target type
             7    17    20   any
            20    23    20   any
      LineNumberTable:
        line 22: 0
        line 23: 7
        line 24: 15
        line 25: 25
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      26     0  this   Lcom/gjxaiou/synchronize/MyTest1;
}
SourceFile: "MyTest1.java"
```



#### 2.synchronized 关键字修饰方法

**总结**：synchronized 关键字修饰方法的时候，并没有出现 monitorenter 和 monitorexit 指令，而是在 JVM 层面通过方法的标识（flags)来标识这个是否为一个同步方法。从字节码中可以看出对应于：`ACC_SYNCHRONIZED`，当线程去调用该方法的时候首先就会去检验该方法是否含有 `ACC_SYNCHRONIZED` 标志位，如果有的话会该执行线程尝试获取当前方法所在的对象的锁（即对象的 monitor 对象），获取到锁之后才会正常的执行方法体。在该方法执行期间，其他任何线程均无法再获取到这个 monitor 对象（锁），当线程执行完方法之后或者抛出异常，它会释放掉这个 monitor 对象。

**测试代码 1**：

```java
package com.gjxaiou.synchronize;

/**
 * @Author GJXAIOU
 * @Date 2020/2/17 20:51
 */
public class MyTest2 {
    public synchronized void method() {
        System.out.println("hello world");
    }
}

```

对应反编译结果

```java
E:\Program\Java\Project\JavaConcurrency\ProficientInJavaConcurrency\target\classes\com\gjxai
ou\synchronize>javap -v MyTest2.class
Classfile /E:/Program/Java/Project/JavaConcurrency/ProficientInJavaConcurrency/target/classe
s/com/gjxaiou/synchronize/MyTest2.class
  Last modified 2020-2-17; size 520 bytes
  MD5 checksum e31ae67e69b765315b9facb2b3d2e48d
  Compiled from "MyTest2.java"
public class com.gjxaiou.synchronize.MyTest2
  minor version: 0
  major version: 49
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
// 常量池省略
{
  public com.gjxaiou.synchronize.MyTest2();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/gjxaiou/synchronize/MyTest2;

  public synchronized void method();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED  // 通过 ACC_SYNCHRONIZED 标识为一个同步方法
    Code:
      stack=2, locals=1, args_size=1
      0: getstatic     #2    // Field java/lang/System.out:Ljava/io/PrintStream;
      3: ldc           #3                  // String hello world
      5: invokevirtual #4 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      8: return
     LineNumberTable:
        line 9: 0
        line 10: 8
     LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lcom/gjxaiou/synchronize/MyTest2;
}
SourceFile: "MyTest2.java"

```



#### 3. synchronized 关键字修饰静态方法

同样通过 ACC_SYNCHRONIZED 标识

当线程去调用该方法的时候首先就会去检验该方法是否含有 `ACC_SYNCHRONIZED`  和 `ACC_STATIC`标志位，如果有的话会该执行线程尝试获取当前方法所在的类的对应 Class 对象的锁（即 Class 对象的 monitor 对象），获取到锁之后才会正常的执行方法体。在该方法执行期间，其他任何线程均无法再获取到这个 monitor 对象（锁），当线程执行完方法之后或者抛出异常，它会释放掉这个 monitor 对象。

**测试程序 1：**

```java
package com.gjxaiou.synchronize;

/**
 * @Author GJXAIOU
 * @Date 2020/2/17 21:12
 */
public class MyTest3 {
    public static synchronized void method() {
        System.out.println("hello world");
    }
}

```

对应的字节码为：

```java
E:\Program\Java\Project\JavaConcurrency\ProficientInJavaConcurrency\target\classes\com\gjxai
ou\synchronize>javap -v MyTest3.class
Classfile /E:/Program/Java/Project/JavaConcurrency/ProficientInJavaConcurrency/target/classe
s/com/gjxaiou/synchronize/MyTest3.class
  Last modified 2020-2-17; size 502 bytes
  MD5 checksum 02b9ebc1b02bd60130e038ace131515a
  Compiled from "MyTest3.java"
public class com.gjxaiou.synchronize.MyTest3
  minor version: 0
  major version: 49
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   // 省略常量池
{
  public com.gjxaiou.synchronize.MyTest3();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/gjxaiou/synchronize/MyTest3;

  public static synchronized void method();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
    Code:
      // 这是显示方法参数个数为 0，因为该方法是静态方法，原则上这个方法并不属于 MyTest3 这个类（我们可以在没有生成 MyTest3 对象的情况下使用类名直接调用该方法），
      stack=2, locals=0, args_size=0
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintSt
ream;
         3: ldc           #3                  // String hello world
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/l
ang/String;)V
         8: return
      LineNumberTable:
        line 9: 0
        line 10: 8
}
SourceFile: "MyTest3.java"
```



### （四）自旋对于 synchronized 关键字的底层意义与价值分析

> 注：一个线程调用了 wait 方法之后就会进入 waitset 中，当另一个线程调用了 notify/notifyAll 方法之后会将这个线程唤醒，唤醒之后争用锁，争到就执行，没争到就阻塞进入了 EntryList 中；阻塞的线程进入 EntryList 中。

JVM 中的同步是基于进入与退出监视器对象（即 Monitor 对象）（Monitor 也称为 **管程**）来实现的，每个对象实例都会有一个 Monitor 对象， Monitor 对象会和 Java 对象一同创建和销毁， Monitor 对象是由 C++ 来实现的。

当多个线程同时访问一段同步代码时，这些线程会被放入一个 EntryList 集合中，处于阻塞状态的线程都会被放到该列表当中，接下来，当线程获取到对象的 Monitor 时候， Monitor 是依赖于底层操作系统的 matex Lock 来实现互斥的，线程获取 mutex 成功，则会持有该 mutex，这时其他线程就无法在获取到该 mutex。

**所以 synchronized 底层是基于操作系统的互斥锁实现的**。

如果线程调用了 wait 方法，那么该线程就会释放掉所持有的 mutex，并且该线程会进入到 WaitSet （等待集合）中，等待下一次被其他线程调用 notify/notifyAll 唤醒，如果当初线程顺利的执行完毕方法，那么他也会释放所持有的 mutex。

当一个线程调用了 wait 方法，该线程就会进入 waitSet（等待集合）中，等待其他线程调用 notify 或者 notifyAll 方法来唤醒，如果被成功唤醒之后，就会尝试着获取对象的锁，如果成功获取了锁就拥有了该对象的 monitor 对象，然后正常执行。如果没有获得对象的锁，该线程就会进入到 EntryList 中（EntryList 中都是存放着被阻塞的线程，它们等待下次获取对象的 mutex）。

**总结**：同步锁在这种实现方式中，因为 monitor 是依赖于底层的操作系统实现，这样就存在用户态与内核态之间的切换（当线程真正执行业务代码的时候，是处于用户态的状态下，当没有获取到对象的锁而处于阻塞或者等待状态，因为 monitor 是由底层操作系统的 matex Lock 实现，所以就进入了内核态，如果该线程尝试获取对象的锁并且成功获取到的话就又转换为了用户态），所以会增加性能开销。通过对象互斥锁的概念来保证共享数据操作的完整性，每个对象都对应于一个可称为【互斥锁】的标记，这个标记用于保证在任何时刻，只能有一个线程访问该对象。

注：JVM 优化：当一个线程持有对象的锁并且正在执行相应的代码的时候，如果该线程发现该方法很快就执行完成，就不会让另一个等待线程进入内核态，而是让该线程进行自旋（空转等待正在执行的线程释放 monitor 对象）， 自旋占用 CPU 资源。因为该等待线程没有阻塞所有一直处于用户态，省去了用户态和内核态之间切换的时间。如果执行线程执行方法过长，等待线程自旋一段时间还是会转入内核态。

那些处于 EntryList 与 waitSet 中的线程均处于阻塞状态，阻塞操作是有操作系统来完成的，在 Linux 中是通过 pthread_mutex_lock 函数实现的。线程被阻塞之后就会进入到内核调度状态，这会导致系统在用户态和内核态之间来回切换，严重影响锁的性能。

解决上述问题的方法便是**自旋**（Spin）。其原理是：当发生对 monitor 的争用的时候，若拥有者能够在很短的时间内释放掉锁，则那些正在争用的线程就可以稍微等待（即所谓的自旋），在拥有者线程释放锁之后，争用线程可能会立即获取到锁，从而避免了系统阻塞。如果拥有者运行的时间超过了临界值之后，争用线程自旋一段时间后依然无法获取到锁，这是争用线程就会停止自旋而进入阻塞状态，所有总体思想是：先自旋，不成功再进行阻塞，尽量降低阻塞的可能性，这对那些执行时间很短的代码块来说有极大的性能提升。显然，自旋在多处理器（多核心）上才有意义。

