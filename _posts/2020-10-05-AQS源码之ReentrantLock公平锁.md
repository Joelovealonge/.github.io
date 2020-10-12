---
title: AQS源码之ReentrantLock公平锁
tags:

  - Java并发
---

### AQS 框架

Java并发编程核心在java.concurrent.util包，而juc当中的大部分同步是实现都是围绕着共同的基础行为，比如等待队列、条件队列、独占获取、共同获取等，而这个行为的抽象就是基于AbstractQueuedSynchronizer简称AQS，ASQ定义了一套多线程访问共享资源的同步器框架，是一个依赖状态（state）的同步器。

**AQS具备特性：**

- 阻塞等待队列
- 共享/独占
- 公平/非公平
- 可重入
- 允许中断

例如Java.concurrent.util当中同步器的实现如Lock，Latch，Barrier等，都是基于AQS框架实现

- 一般通过定于内部类Sync继承AQS

- 将同步器所有调用都映射到Sync对应的方法，AQS内部维护属性volatile int state（32位）

- state表示资源的可用状态

- State三种访问方式

  getState()、setState()、compareAndSetState()

- AQS定义两种资源共享方式

  - Exclusive-独占，只有一个线程能执行，入ReentrantLock
  - Share-共享，多个线程可以同时调用，如Semaphore/CountDownLatch

- AQS定义两种队列

  - 同步等待队列
  - 条件等待队列

不同的自定义同步器争用共享资源的方式也不同。**自定义同步器在实现是只需要实现共享资源state的获取与释放方式即可**，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队列等），AQS已经在顶层实现好了。

自定义同步器实现是主要实现以下几个方法：

| 方法                  | 方法含义                                                     |
| --------------------- | ------------------------------------------------------------ |
| isHeldExclusively()   | 该线程是否正在独占资源。只有用到condition才需要去实现它。    |
| tryAcquire(int)       | 独占方式。尝试获取资源，成功则返回true，失败则返回false。    |
| tryRelease(int)       | 独占方式。尝试释放资源，成功则返回true，失败则返回false。    |
| tryAcquireShared(int) | 共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。 |
| tryReleaseShared(int) | 共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。 |

它维护了一个volatile int state（代表共享资源）和一个CHL队列。

state访问的三种方式：

- getState()
- setState()
- compareAndSetState()

AQS定义了两种资源共享方式：

- Exclusive 独占

  如ReentrantLock

- Share 共享

  如Semaphore/CountDownLatch
  
  

AQS其底层采用乐观锁，大量使用了CAS操作，并在冲突时，采用自旋方式重试，以实现轻量级和高效地获取锁。

AQS虽然被定义为抽象类，但事实上它并不包含任何抽象方法。这是因为AQS是被设计来支持多种用途的，如果定义抽象方法，则子类在继承时必须要覆写所有抽象方法，这显然是不合理的。所以AQS将一些需要子类覆写的方法都设计成protect方法，将其默认实现为抛出`UnsupportedOperationException`异常。如果子类使用到这些方法，但是没有覆写，则会抛出异常；如果子类没有使用这些方法，则不需要做任何操作。



### AQS的三板斧

- 状态

  **state属性**，是并发工具的核心，通常整个工具都是在**设置修改状态**。由于状态时全局共享的的，一般会被设置成volalite类型，保证其修改的可见性。

- 队列

  队列是一个等待的集合，大多数以链表的形式实现。**队列采用悲观锁**的思想，表示当前锁等待的资源，状态或者条件短时间内可能无法满足。因此，它会**将当前线程包装成某种类型的数据结构**，扔到一个等待队列中，当一定条件满足后，再从等待队列中取出。

- CAS

  CAS操作是最轻量的并发处理，通常我们对于**状态的修改都会用到CAS操作**，因为状态可能被多个线程同时修改，CAS操作保证了同一时刻，只有一个线程能修改成功，从而保证了线程安全，CAS操作基本是由Unsafe工具类的`compareAndSwapXXX`来实现的；**CAS采用的是乐观锁**的思想，因此常常伴随着自旋，如果发现当前**无法成功地执行CAS，则不断重试**，直到成功为止，自旋的表现形式通常是一个死循环`for(;;)`。

### AQS源码详解

**状态**

```java
private volatile int state;
```

该属性值表示了锁的状态，0表示没有被占用，大于0表示已被占用。

```java
// 继承自AbstractOwnableSynchronizer
private transient Thread exclusiveOwnerThread;
```

该属性表示当前持有锁的线程。

note：java transient 关键词表示序列化的时候，该属性不需要被序列化。

**队列**

双向链表实现，表示**所有等待锁的线程的集合**，被称为`sync queue`。

队列中的节点：

```java
static final class Node {

        static final Node SHARED = new Node();

        static final Node EXCLUSIVE = null;
    
		// 线程锁处的等待锁的状态，初始化时，该值为0
    	volatile int waitStatus;
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;

        
		// 双向链表，每个节点需要保存自己前驱结点和后继结点的引用
        volatile Node prev;
        volatile Node next;

    	// 结点锁代表的线程
        volatile Thread thread;

    	// 该属性用于条件队列或者共享锁
        Node nextWaiter;

        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

在这个静态内部类Node类中有一个状态变量`waitStatus`，表示当前Node所代表的线程等待锁的状态，独占模式下，我们只需要关注`CANCELLED、SIGNAL`两种状态即可。`nextWaiter`属性，在独占模式下永远为null，仅仅起到一个标记作用。

那么AQS如何使用这个队列呢，两个节点：

```java
    // 头结点，不代表任何线程，是一个哑节点
	private transient volatile Node head;

	// 尾结点，抢占锁失败后，排队到队尾
    private transient volatile Node tail;
```

sync queue全貌，是一个FIFO的CLH队列。Java中的CLH队列是原CLH队列的一个变种，线程由原自旋机制改为阻塞机制。

![preview](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/1460000016446954)

- thread：表示当前node所代表的线程
- waitStatus：表示结点所处的等待状态，独占模式下只需要关注三种状态：`SIGNAL、CANCELLED、初始状态（0）`
- prev next：节点的前驱和后继
- nextWaiter：用作标记，独占模式下永远为null



**CAS操作**

CAS操作大多数是用来改变状态的，而Unsafe类通过对象属性的偏移定位内存中的位置，我们一般在静态代码块中初始化需要CAS操作的属性的偏移量：

```java
private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long waitStatusOffset;
    private static final long nextOffset;

    static {
        try {
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
            waitStatusOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("waitStatus"));
            nextOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("next"));

        } catch (Exception ex) { throw new Error(ex); }
    }
```

可以看到CAS操作主要针对5个属性，包括AQS的3个属性`state,head和tail`以及node对象的两个属性`waitStatus,next`。说明这5个属性是会被多线程同时访问的。

CAS操作本身：

```java
    private final boolean compareAndSetHead(Node update) {
        return unsafe.compareAndSwapObject(this, headOffset, null, update);
    }

    private final boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }

    private static final boolean compareAndSetWaitStatus(Node node,
                                                         int expect,
                                                         int update) {
        return unsafe.compareAndSwapInt(node, waitStatusOffset,
                                        expect, update);
    }

    private static final boolean compareAndSetNext(Node node,
                                                   Node expect,
                                                   Node update) {
        return unsafe.compareAndSwapObject(node, nextOffset, expect, update);
    }
```



### 一、ReentrantLock锁的获取源码分析

前面已经提到AQS大多数情况下都是通过继承子类来实现的，子类通过覆写`tryAcquire`来实现自己获取锁的逻辑。

ReentrantLock有公平锁和非公平锁两种实现方式，我们看源码可以知道**默认实现为非公平锁**

```java
public class ReentrantLock implements Lock, java.io.Serializable {

    private final Sync sync;

    abstract static class Sync extends AbstractQueuedSynchronizer {
    	...
    }

    static final class NonfairSync extends Sync {
    	...
    }
    
    static final class FairSync extends Sync {
    	...  
    }

	 // 默认构造函数
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

    public void lock() {
        sync.lock();
    }
}
```

可以看出FairSync继承自Sync，而Sync继承自AQS，ReentrantLock获取锁的逻辑直接调用了FairSync或者NonfairSync的逻辑。

这里我们以FairLock来分析锁的获取：

```
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;
		// 获取锁
        final void lock() {
            acquire(1);
        }
        ...
	}
```

lock方法调用的acquire方法来自父类AQS。

完整的获取锁流程图：

![image-20201005152412042](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201005152412042.png)

**acquire**

acquire定义在AQS类中，描述了获取锁的流程

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

1. tryAcquire(arg)

   该方法由继承AQS的子类实现，为获取锁的具体逻辑。

2. addWaiter(Node mode)

   该方法由AQS实现，负责获取锁失败后调用，将当前锁的线程包装成Node扔到同步队列（sync queue）中去，并返回这个Node。

3. acquireQueued(final Node, int arg)

   该方法由AQS实现，主要对上面加入队列的Node不断尝试一下两种操作之一：

   - 在前驱结点就是head结点的时候，继续尝试获取锁
   - 将当前线程挂起，使CPU不在调度它

4. selfInterrupt

   该方法由AQS实现，用于中断当前线程。由于在整个抢锁过程中，我们都是不响应中断的。那如果在抢锁的过程中发生了中断怎么办呢，总不能假装没看见呀。AQS的做法是简单的记录有没有发生过中断，如果返回的时候发现曾经发生过中断，则在退出acquire方法之前，就调用selfInterrupt自我中断一下，延迟中断。

接下来我们重点分析一下FairSync所实现的获取锁的逻辑：

**tryAcquire**

判断当前锁有没有被占用：

 	1. 如果锁没有被占用，尝试以公共的方式获取锁
 	2. 如果锁已经被占用，检查是不是锁重入

获取锁成功返回true，失败则返回false

```java
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            // 获取锁的当前状态
            int c = getState();
            // c=0, 说明当前锁没有被任何线程占用，可以尝试获取
            // 因为是实现公平锁，所以在抢占之前先看看队列中有没有已经在排队的线程（Node）
            // 如果没有人排队，则通过CAS方式获取锁，就可以直接退出了
            if (c == 0) {
                if (!hasQueuedPredecessors() 
                    /*  判断同步队列中第一个node是不是自己，是则返回true，不是则返回false
                    	public final boolean hasQueuedPredecessors() {
                            Node t = tail; 
                            Node h = head;
                            Node s;
                            return h != t &&
                                ((s = h.next) == null || s.thread != Thread.currentThread());
                        }
                    */
                    && compareAndSetState(0, acquires)) {
                    // 将当前线程设置为锁占用线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 如果c>0 说明锁被占用了
            // 对于可重入锁，这个时候检查占用锁的线程是不是当前线程，是的话直接重入就行了
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                // 这儿因为已经取得了锁，所以不需要cas设置，用普通方法设置就可以
                setState(nextc);
                return true;
            }
            // 到这里说明有人占用了锁，并且占用锁的不是当前线程，即获取锁失败
            return false;
        }
```



**addWaiter**

如果执行到此方法，说明前面尝试获取锁的tryAcquire已经失败了，既然已经失败了，就要将当前线程包装成Node，加到等待锁的同步队列中，加到队尾。

```java
addWaiter(Node.EXCLUSIVE)
```

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
		/*
			Node.EXCLUSIVE 为null
			
			Node(Thread thread, Node mode) {     
                this.nextWaiter = mode;
                this.thread = thread;
            }
		*/
        Node pred = tail;
        // 如果队列不为空，则用CAS方式将当前节点设为尾结点，为空则表示head结点都没有
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // 代码会执行到这里，只有两种情况
        //		1. 队列为空
        //		2. CAS失败，这里是并发条件下，所以cas可能会失败
        enq(node); // 将节点插入队列
        return node;
    }
```

可见，每一个处于独占模式下的节点，它的nextWaiter一定为null。

在这个方法中，我们首先尝试直接入队，但是因为目前在并发条件下，所以有可能同一时刻，有多个线程都在尝试入队，导致`compareAndSetTail(pred, node)`操作失败------因为有可能其他线程已经成为了新的尾结点，导致尾结点不再是我们之前看到的那个pred了。



如果入队失败了，接下来我们就需要调用enq(node)方法，在该方法中我们将通过**自旋+CAS**的方式，保证当前结点入队。



**enq**

能执行到这个方法，说明当前线程获取锁已经失败了，我们已经把它包装成一个Node，准备把它扔到等待队列中去，但是在这一步又失败了。这个失败的原因可能是以下两种之一：

1. 等待队列现在是空，没有线程在等待。
2. 其他线程在当前线程入队的过程中率先完成了入队，导致尾结点的值已经改变了，CAS操作实拍。

在该方法中，我们使用了死循环，即以自旋方式将节点插入队列，如果失败则不停的尝试，知道成功为止，另外，该方法也负责在队列为空时，初始化队列，这也说明，队列是延时初始化的：

```java
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            // 如果是空队列，首先进行初始化
            if (t == null) { 
                // 初始化时使用new Node() 新建了一个结点
                if (compareAndSetHead(new Node()))
                    // 将尾结点指向head结点
                    tail = head;
            } else {
            // 到这里说明队列已经不是空了，继续尝试将节点加到队尾
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    // 返回当前节点的前驱结点
                    return t;
                }
            }
        }
    }
```

**尾分叉现象**

enq方法中将当前节点设置成尾结点的操作：

```java
      } else {
           node.prev = t;
           if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
```

将一个结点node添加到同步等待队列的尾部需要三步：

	1. 设置node的前驱结点为当前的尾结点：`node.prev = t`
 	2. 修改tail属性，使它指向当前结点
 	3. 修改原来的尾结点，使它的next指向当前结点

但是在并发场景下，第一步很容易成功，第二步可能失败，第二步失败后第三步不能执行。

也就是大量线程同时入队的时候，只有一个线程能完整地完成这三步，**而其他线程只能完成第一步**，于是出现尾分叉：

![preview](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/view)

注意：有可能我们已经完成了第一二步，将新的结点设置成尾结点，**此时原来的旧的next值可能还没来得及执行第三步即值还为null**。此时如果有线程恰巧从头结点向后遍历整个链表，则它此时是遍历不到新加进来的尾结点的。这也是为什么我们在AQS相关的源码中，有时候常常会从尾结点开始逆向遍历链表。

至于那些分叉的入队失败的其他结点，在一下轮的循环中，他们的prev属性会重新指向新的尾结点，继续尝试新的CAS操作，最终，所有结点都会通过自旋不断的尝试入队，直到成功为止。



**addWaiter总结**

该方法并不涉及任何关于锁的操作，他解决了并发情况下结点入队问题，具体来说就是该方法保证了将当前线程包装成Node结点加到等待队列的队尾，如果队列为空，则会新建一个哑结点作为头结点，再讲当前结点接在头结点的后面。

addWaiter(Node.EXCLUSIVE)方法最终返回了代表当前线程的Node结点，在返回的那一刻，这个节点必然是当时队列的尾结点。



enq方法也有返回值（虽然这里我们并没有使用它的返回值），但是它返回的是node节点的前驱节点。

我们再回到获取锁的逻辑中：

```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

当addWaiter(Node.EXCLUSIVE)执行完毕后，节点现在已经被成功添加到同步等待队列中了，接下来将执行acquireQueued方法。



**acquireQueued**

最复杂的一个方法，

1. 能执行到该方法，说明addWaiter方法已经成功将包装了当前thread的结点添加到了等待队列的队尾
2. 将方法中将再次去获取锁
3. 在再次尝试获取锁失败后，判断是否需要把当前线程挂起

**为什么前面获取锁失败了，这里还要再次尝试获取锁呢？**

首先这里再次尝试获取锁是基于一定的条件的，即**`当前节点的前驱结点就是HEAD结点`**，因为第一次获取锁失败可能是因为同步队列并没有初始化即head结点都没。

```java
	final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                // 当前结点的前驱就是head结点时，再次尝试获取锁
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 在获取锁失败后，判断是否需要把当前线程挂起
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

获取锁成功后，我们可以看到当前结点（获取到锁的结点）转换为head哑结点，丢弃之前的head结点。



**shouldParkAfterFailedAcquire**

该方法用于决定获取锁失败后，是否将线程挂起。

决定的依据是前驱节点的**waitStatus**值。

我们先回顾一下waitStatus的几种取值吧：

```java
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;
```

一共五种取值，0位初始值。

CANCELLED状态表示了Node所代表的当前线程已经取消了排队，即放弃获取锁了。

重点需要知道SIGNAL状态。

如果一个结点的waitStatus的属性为SIGNAL，那么当他释放锁或者放弃锁后，需要唤醒它的后继结点（线程）。

注意：ws > 0 表示僵尸结点，即不正常的结点。

```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        // 前驱结点的状态已经是SIGNAL，说明闹钟已经设。可以直接睡了即可以阻塞了
        if (ws == Node.SIGNAL)
            return true;
        // 依次向前找到离当前结点最近的一个正常的结点，找到后跳出循环
        if (ws > 0) {
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

可以看出，shouldeParkAfterFailedAcquire所做的事情如下：

- 如果当前前驱结点的waitStatus值为Node.SIGNAL则直接返回true
- 如果前驱节点的waitStatus值为Node.CANCELLED,则跳过那些结点，重新寻找正常等待中的前驱节点，然后排在它后面，返回false
- 其他情况，将前驱结点的状态改为Node.SIGNAL,返回false



我们在回到这个方法的调用出，可以看出当shouldParkAfterFailedAcquire返回false后，会继续尝试获取锁，因为此时我们的前驱节点有可能变成了head结点了，有可能直接能获取到锁。



当shouldParkAfterFailedAcquire返回true，即当前节点的前驱结点waitStatus状态已经被设置成SIGNAL后，我们就可以安心的将线程挂起了，此时我们将调用parkAndCheckInterrupt:



**parkAndCheckInterrupt**

到这个函数已经是最后一步了，就是将线程挂起，等待被唤醒

```java
    private final boolean parkAndCheckInterrupt() {
        // 线程被挂起，停在这里不再往下执行了
        LockSupport.park(this);
        return Thread.interrupted();
    }
```

注意！LockSupport.park(this)执行完成后线程就被挂起了，除非其他线程unpark了当前线程，或者当前线程被中断了，否则代码是不会再往下执行的，后面的Thread.interruted()也不会被执行，那后面这个Thread.interrupted()是干什么用的呢？



### 二、ReentrantLock锁的释放源码分析

java内置锁在退出临界区之后是会自动释放锁的，但是ReentrantLock这样的显示锁是需要自己显示释放的，所以在加锁之后一定要在finally块进行显示的锁释放

```
Lock lock = new ReentrantLock();
...
lock.lock();
try{
	
} finally {
	lock.unlock();
}
```

由于锁的释放操作对于公平锁和非公平锁都是一样的，所以，unlock的逻辑并没有放在FairSync或NonfairSync里面，而是直接定义在ReentrantLock类中：

```java
    public void unlock() {
        sync.release(1);
    }
```



**release**

release方法定义在AQS类中，描述了释放锁的流程

```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

- tryRelease(arg)

  该方法由继承AQS的子类实现，为释放锁的具体逻辑

- unparkSuccessor(h)

  唤醒后继线程



**tryRelease**

tryRelease方法由ReentrantLock的静态类Sync实现：

```java
        protected final boolean tryRelease(int releases) {
            // 首先将当前持有锁的线程个数减1
            // 这里的操作主要是针对可重入锁的情况下，c可能大于1
            int c = getState() - releases;
            
            // 释放锁的线程必须是持有锁的线程
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            
            // 如果c为0了，说明锁已经完全释放了
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

锁成功释放后，接下来就是唤醒后继结点了，这个方法unparkSuccessor同样定义在AQS中

**unparkSuccessor**

```java
    private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;
        // 如果head结点的ws比0小，则直接将它设为0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        // 通常情况下，要唤醒的结点就是自己的后继结点
        // 如果后继结点存在且也在等待锁，那就直接唤醒它
        // 但是也有可能存在 后继结点取消等待锁的情况
        // 此时从尾结点开始向前找起，知道找到距离head结点最近的ws<0的结点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        // 如果找到了还在等待锁的结点，则唤醒它
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

最后，在调用了LockSupport.unpark(s.thread)也就是唤醒线程之后，会发生什么呢？

当然是回到最初的原点啦

```java
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        // 这里
        return Thread.interrupted();
    }
```

线程从这里唤醒，它将接着这里往下执行。

我们知道，Thread.interrupted()这个函数将返回当前正在执行的线程的中断状态，并清除它。接着，我们再返回到parkAndCheckInterrupt被调用的地方：

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 在这里哦
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

可见，如果Thread.interrupted()返回true，则parkAndCheckInterrupt()就返回true，if条件成立，interrupted状态将设为true；

若返回false，则interrupted状态将设为false。



再接下来我们又回到了for(;;)死循环的开头，进行新一轮的抢锁。

假如我们这次抢到了，我们将从return interrupted处返回，返回acquireQueued的调用处。

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

我们可以看到，如果acquireQueued的返回值为true，我们将执行selfInterrupt()。

绕了这么一大圈，到最后还是中断了当前线程，到底是在干嘛呢？

其实这一切的原因在于：**我们并不知道线程被唤醒的原因。**

具体来说，当我们从LockSupport.park(this)处被唤醒，我们并不知道是因为什么原因被唤醒，可能是因为别的线程释放了锁，调用了LockSupport.unpark(s.thread)，也可能是因为当前线程在等待中被中断了，因此我们通过Thread.interrupter()方法检查了当前线程的中断标志，并将它记录下来，在我们最后返回acquire方法后，如果发现当前线程曾经被中断过，那我们就把当前线程再中断一次。

为什么要这么做呢？

从源码中我们可以知道，即使线程在等待资源的过程中被中断唤醒，它还是会不断的抢锁，知道它抢到锁为止。也就是说，它是不响应这个中断的，仅仅是记录下自己被人中断过。



最后它抢到锁返回了，如果它发现自己曾经被中断过，它就再中断一次，将这个中断补上。



注意：中断对线程来说只是一个建议，一个线程被中断只是其中断状态被设为true，线程可以选择忽略这个中断，一个中断线程并不会影响线程的执行。



那后面的finally是干啥用的呢？

跳出for循环并不一定拿到了锁，也可能是tryAcquire(arg), 我们并不知道子类如何去实现他，一旦子类获取锁抛出异常，就必须要走finally的cancelAcquire(node)方法，将本节点从同步队列中移除。





**彩蛋1：finally块中的代码什么时候被执行?**

在Java语言的异常处理中，finally块的作用就是为了保证无论出现什么情况，finally块里的代码一定会被执行。由于程序执行return就意味着结束对当前函数的调用并跳出这个函数体，**因此任何语句要执行都只能在return前执行（除非碰到exit函数），因此finally块里的代码也是在return之前执行的**。此外，如果try-finally或者catch-finally中都有return，那么finally块中的return将会覆盖别处的return语句，最终返回到调用者那里的是finally中return的值

当finally块中有return语句时，将会覆盖函数中其他return语句。此外，由于在一个方法内部定义的变量都存储在栈中，当这个函数结束后，其对应的栈就会被回收，此时在其方法体中定义的变量将不存在了，因此，对基本类型的数据，在finally块中改变return的值对返回值没有任何影响，而对引用类型的数据会有影响

程序在执行到return时会首先将返回值存储在一个指定的位置，其次去执行finally块，最后再返回。在方法testFinally1中调用return前，先把result的值1存储在一个指定的位置，然后再去执行finally块中的代码，此时修改result的值将不会影响到程序的返回结果。testFinally2中，在调用return前先把s存储到一个指定的位置，由于s为引用类型，因此在finally中修改s将会修改程序的返回结果。



那么什么时候finally快不会被执行呢？

1. 当程序进入try块之前就出现异常时，会直接结束，不会执行finally块中的代码。
2. 当程序在try块中强制退出（System.exit(0)）也不会去执行finally块



