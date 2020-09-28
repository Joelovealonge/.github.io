---
title: synchronized
tags:

  - Java并发
---



多线程编程中，有可能出现多个线程同时访问用一个共享、可变资源的情况，这种资源可能是：对象、变量、文件等。

- 共享：

  资源可以由多个线程同时访问

- 可变：

  资源可以在其生命周期内被修改



### java锁

![image-20200928114024554](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20200928114024554.png)

共享资源又称为临界资源

加锁目的，序列化访问临界资源，同步互斥访问临界资源。

加锁可以保证代码块同一时间只有一个线程执行，但是代码块里面是可能会发生指令重排的。

加锁有两种锁：

- 显示锁

  ReentrantLock，实现juc里Lock，是基于AQS实现，需要手动加锁和解锁ReentrantLock lock(), unlock()； 灵活性高

- 隐示锁

  synchronized加锁机制，JVM内置锁，不需要手动加锁与解锁，Jvm会自动加锁解锁，几乎不可以跨方法，灵活性低。

  加锁粒度是对象-----对象锁。
  
  
### synchronized

  synchronized内置锁是一种对象锁（锁的是对象而非引用），作用粒度是对象，可以用来实现对临界资源的同步互斥访问，是可重入的。

加锁方式：

- 同步实例方法，锁是当前实例对象

  synchronized method()   和  synchronized(this){} 这两种方法只是写法不同

- 同步类方法，锁的是当前类对象

  static synchronized method() 

  需要注意的是：类中两个不同方法都加了static synchronized ，但是线程就会互斥访问。

- 同步代码块，锁的是括号里的对象

  synchronized(object){}



**底层原理：**

synchronized是基于JVM内置锁实现，通过内部对象Monitor（监视器锁）实现，基于进入与退出Monitor对象实现方法与代码块同步，监视器锁的实现依赖底层操作系统的Mutex lock（互斥锁）实现，它是一个重量级锁性能比较低。

synchronized 关键字被编译成字节码文件后会翻译成两对指令monitorenter和monitorexit，分别加到了代码块的开始位置和结束位置。

monitorenter  底层逻辑其实基于jmm的一个原子操作lock

![image-20200928202059081](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20200928202059081.png)



每个同步对象都有一个自己的Monitor（监视器锁），加锁过程如下图所示：

![image-20200928181304670](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20200928181304670.png)



那么对象是如何记录**锁状态**的呢？

锁状态是被记录在每个对象的对象头（Mark Word）中，

**对象的内存的分布：**

![image-20200928181725604](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20200928181725604.png)

- 对象头：

  比如hash码，对象所处年代，对象锁，锁状态标志，偏向锁（线程）ID，偏向时间，数组长度（数组对象）等。

- 实例数据：

  即创建对象时，对象成员变量，方法等。

- 对齐填充：

  对象的大小必须是8字节的整数倍。



Mark Word会随着程序的运行发生变化，以32位虚拟机为例，说一下对象头信息的变化（64位虚拟机对象头存储的信息一样，但是位数和位置不一样）

![image-20200928202159990](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20200928202159990.png)

JVM内置锁在jdk1.6以后做了优化，并发性能已经基本与Lock持平。如锁粗化、锁清除、轻量级锁、偏向锁、适应性自旋等。



**锁粗化：**

就比如 StringBuffer 是线程安全，我们看源码可以知道StringBuffer 在append()方法上加了对象锁。

当我们追加数据的时候：

```java
public void test() {
        sb.append("1");
        sb.append("2");
    }
```

就相当于：

```java
public class SynchronizedDemo {
    StringBuffer sb = new StringBuffer();
    
    public void test1(){
        synchronized (this){
            sb.append("1");
        }
        synchronized (this){
            sb.append("2");
        }
    }
}
```

但是JVM进行锁粗化：

```java
    public void test2() {
        synchronized (this) {
            sb.append("1");
            sb.append("2");
        }
    }
```



**额外加餐：** 实例对象存在内存中哪里？

如果实例对象存储在堆区时：实例对象内存存在’堆区，实例对象的引用存在栈上，实例对象的元数据class存在方法区或者元数据区。

实例对象一定是存在堆区的吗？

不一定，如果实例对象很简单而且没有线程逃逸行为，可能直接存在线程栈空间。

Jit编译时会代码进行逃逸分析



**逃逸分析：**

 使用逃逸分析，编译器可以对代码做如下优化：

- 同步省略。如果一个对象被发现只能从一个线程被访问到，那么对于这个对象的可以不考虑同步。
- 将堆分配转化为栈分配，如果一个对象在子程序中别分配，要使指向该对象的指针永远不会被逃逸，对象可能是栈分配候选，而不是堆分配。
- 分离对象或标量替换。有的对象可能不需要作为一个连续的内存结构存在也可以被访问到，那么对象的部分（或全部）可以不存储在内存，而是存储在CPU寄存器中。

java1.7之后默认开启逃逸分析进行优化

开启逃逸分析：vm运行参数

`-Xmx4G -Xms4G -XX:+DoEscapeAnalysis -xx:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError`

关闭逃逸分析：同时调大堆空间，避免堆内GC的发生

``-Xmx4G -Xms4G -XX:-DoEscapeAnalysis -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError``



**偏向锁：**

 偏向锁是java1.6之后加入的，是一种对加锁操作的优化手段，经过研究发现，在大多数情况下，锁直接不仅不存在多线程竞争，而且总是由同一线程多次获得，因此为了减少同一线程获取锁（会涉及到一些CAS操作，耗时）的代价而引入偏向锁。

核心思想是：如果一个线程获得了锁，那么锁就进入偏向模式，此时Mark Word的结构也变成偏向锁结构，当这个线程再次请求锁时，无需在做任何同步操作，即获取锁的过程，这样就省去了大量有关锁申请的操作，从而提供性能。偏向锁不会主动释放。

**轻量锁：**

当申请锁的线程不同时，极有可能偏向锁失败，虚拟机会尝试轻量锁。

轻量锁提高性能的依据是：对绝大部分的锁，在整个同步周期内都不存竞争，轻量锁适应的场景是线程交替同步执行同步块的场合，如果存在同一时间访问同一锁的场合，就会导致轻量锁膨胀为重量锁。



**锁自旋：**

轻量锁失败后，虚拟机为了避免线程真实地在系统层面被挂起，采用的一种自旋锁的优化手段。这是基于绝大多数情况下，线程持有锁的时间都不会太长。但是系统线程直接的切换需要用户态和内核态的转换，上下文切换等时间成本。最后没办法就装换成重量锁。

![image-20200928183423623](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20200928183423623.png)

进程自旋不会放弃CPU使用权，不需要阻塞，因此避免了进程的上下文切换。

java1.7 之后 自旋次数，根据上次自旋成功次数，适当调整本次自旋次数。



**锁清除：**

这种优化更彻底，java虚拟机在JIT编译时，通过对上下文扫描，去除不可能存在的资源竞争的锁，通过这种方式清除没有必要的锁。比如StringBuffer的append()是一个同步方法，但是在add方法中的StringBuffer属于一个局部变量，并且不会被其他进程所使用，因此StringBuffer不可能存在共享资源竞争的情况，JVM自动将其锁清除。

```java
    public void test3() {
        // jvm 的优化
        synchronized(new Object()) {
            // 伪代码 很多逻辑
        }
    }
```



**锁的膨胀升级**

![image-20200928200436918](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20200928200436918.png)

![image-20200928200504332](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20200928200504332.png)

synchronized锁实现与升级过程

![image-20200928200722660](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20200928200722660.png)



