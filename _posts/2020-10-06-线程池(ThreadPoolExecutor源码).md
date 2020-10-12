---
title: 线程池(ThreadPoolExecutor源码)
tags:

  - Java并发
---



### 线程池

什么时候使用线程池：

- 单个任务处理时间比较短
- 需要处理的任务数量很大



线程池的优势：

- **重用存在的线程，减少线程的创建，消亡开销，提高性能。**
- 提高响应速度，当任务到达时，任务可以不需要等到线程创建就能立即执行。
- 提供线程的可管理性。线程是稀缺资源，如果被无限制创建，不仅会消耗资源，还会降低系统的稳定性，使用线程池可以线程进行统一分配、调优和监控。



**线程的实现方式**

Runnable，Thread，Callable

我们主要来看一下Callbale，其他两种实现方式详见Thread类详解

```java
// Callable同样是任务，与Runnable接口的区别在于它能接收泛型，同时它执行任务后带有返回内容
public interface Callable<V> {
    // 相对于run方法的带有返回值的call方法
    V call() throws Exception;
}
```



### Executor框架

Executor接口是线程池中最基础的部分，定义了一个用于执行Runnable的execute方法。

```java
public interface Executor {
    void execute(Runnable command);
}
```

下图为它的继承与实现：

![image-20201006143728829](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201006143728829.png)

可以看出Executor下有一个重要的子接口ExecutorService，其中定义了线程池的具体行为

```java
public interface ExecutorService extends Executor {

    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();

    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

1. execute(Runnable command)

   履行Runnable类型的任务。

2. submit(task)

   可用来提交Callable或Runnable任务，并返回代表此任务执行的Future对象。

3. shutdown()

   在完成已提交的任务后封闭办事，不在接管新任务。会等待正在执行的任务和等待队列中没执行的任务全部执行完毕。

4. shutdownNow()

   停止所有正在履行的任务并封闭办事，并返回正在等待队列中的任务列表。

5. isTerminated()

   测试是否所有任务都履行完毕了。

6. isShutdown()

   测试是否该ExecutorService已被关闭。



默认线程池的实现：ThreadPoolExecutor

### ThreadPoolExecutor

线程池重点属性

```java
public class ThreadPoolExecutor extends AbstractExecutorService {    
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    
    private static final int COUNT_BITS = Integer.SIZE - 3;
    
    // COUNT_BITS 是29 
    // 1 << COUNT_BITS 也就是		   001 0 0000 0000 0000 0000 0000 0000 0000
    // (1 << COUNT_BITS) - 1  也就是  000 1 1111 1111 1111 1111 1111 1111 1111
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // 线程池的5种状态，可以看到用高3位就可以表示
    // 111 0 0000 0000 0000 0000 0000 0000 0000
    private static final int RUNNING    = -1 << COUNT_BITS;
    // 000 0 0000 0000 0000 0000 0000 0000 0000
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    // 001 0 0000 0000 0000 0000 0000 0000 0000
    private static final int STOP       =  1 << COUNT_BITS;
    // 010 0 0000 0000 0000 0000 0000 0000 0000
    private static final int TIDYING    =  2 << COUNT_BITS;
    // 011 0 0000 0000 0000 0000 0000 0000 0000
    private static final int TERMINATED =  3 << COUNT_BITS;

    // ~CAPACITY	111 0 0000 0000 0000 0000 0000 0000 0000
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    // CAPACITY		000 1 1111 1111 1111 1111 1111 1111 1111
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
    
```

ctl是对线程池的运行状态和线程池中有效线程的数量进行控制的一个字段，

它包含两部分信息：

- 线程池的运行状态（runState）
- 线程池内有效线程的数量（WorkerCount）

这里可以看到使用Integer类型报错，高3位保存runState，低29位保存workerCount。



**线程池状态：**

| 线程池状态 | 状态说明                                                     | 状态切换                                                     |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| RUNNING    | 线程池处在RUNNING状态时，能够接收新任务，以及对已添加的任务进行处理。 | 线程池的初始状态是RUNNING。换句话说，线程池一旦被创建，就处于RUNNING状态，并且线程池中的任务数为0 |
| SHUTDOWN   | 线程池处在SHUTDOWN状态时，不接收新任务，但能处理已添加的任务（包括等待队列中的任务）。 | 调用线程池的shutdown()方法时，线程池由RUNNING -> SHUTDOWN    |
| STOP       | 线程池处于STOP状态时，不接收新任务，不处理已经添加的任务，并且会中断正在处理的任务。 | 调用线程池的shutdownNow()方法时，线程池由（RUNNING or SHUTDOWN） -> STOP |
| TIDYING    | 当所有任务已终止，ctl记录的“任务数量”为0，线程池会变成TIDYING状态，当线程池变为TIDYING状态时，会执行狗子函数terminated()。terminated()在ThreadpoolExecutor类中是空的，若用户想在线程池变成TIDYING时，进行相应的处理；通过加载terminated()函数来实现。 | 当线程池在SHUTDOWN状态下，阻塞队列为空并且线程池中执行的任务也为空时，就会由SHUTDOWN -> TIDYING；当线程池在STOP状态下，线程池中执行的任务为空时，就会由STOP -> TIDYING。 |
| TERMINATED | 线程池彻底终止，变成TERMINATED状态。                         | 线程处在TIDYING状态时，执行完TERMINATED()之后，就会由TIDYING -> TERMINATED。 |

进入TERMINATED的条件如下：

- 线程池不是RUNNING状态
- 线程池状态不是TIDYING状态或TERMINATED状态
- 如果线程池状态时SHUTDOWN并且workerQueue为空
- workerCount为0
- 设置TIDYING状态成功

![image-20201006161351406](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201006161351406.png)



**线程池的创建**

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) 
```

核心参数解释：

1. **corePoolSize**

   线程池中核心线程数，当提交一个任务时，线程池创建一个新任务执行，知道当前线程数等于corePoolSize；如果当前线程数为corePoolSize，继续提交的任务被保存到阻塞队列中，等待被执行；如果执行了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有核心线程。

2. **maximumPoolSize**

   线程池中允许的最大线程数。如果当前阻塞队列满了，且继续提交任务，则创建新的线程执行任务，前提是当前线程数小于maximumpoolSize。

3. **keepAliveTime**

   线程池维护线程所允许的空闲时间。当线程池中的数量大于corePoolSize的时候，如果这时没有提交新的任务，核心线程外的**临时线程不会立即销毁，而是会等待，知道等到的时间超过了keepAliveTime**。

4. unit

   keepAliveTime的单位

5. **workQueue**

   用来保存等待被执行的任务的阻塞队列，且任务必须实现Runnable接口，在JDK中提供了如下阻塞队列：

   1. ArrayBlockQueue：基于数组结构的有界阻塞队列，FIFO；
   2. LinkedBlockQueue：基于列表结构的阻塞队列，按照FIFO排序任务,吞吐量通常高于ArrayBlockQueue。
   3. SynchronousQueue：一个不存储元素的阻塞队列，每个插入操作必须的等到另一个线程移除操作；否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedblockingQueue；
   4. priorityBlockingQueue：具有优先级的无界阻塞队列。

6. **threadFactory**

   它是ThreadFactory类型的变量，用来创建新线程。默认使用Executors.defaultThreadFactory()来创建线程。使用默认的ThreadFactory来创建线程时，会使新创建的线程具有相同的NORM_PRIORITY优先级并且是非守护线程，同时也设置了线程的名称。

7. **handle**

   线程池的饱和策略，当阻塞队列满了，且没有空闲的工作线程。如果继续提交任务，必须采取一种策略处理该任务，线程池提供了4中策略：

   1. AbortPolicy：直接抛出异常，默认策略；
   2. CallerRunsPolicy：用调用者所在的线程来执行任务；
   3. DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
   4. DiscardPolicy：直接丢弃任务；

   上述的4中策略都是ThreadPoolExecutor的内部类，实现了RejectedExecutionHandle接口，当然我们可以自定义饱和策略，如记录日志或持久化存储不能处理的任务。

**任务提交**

1. ```java
   public void execute()	// 提交任务无返回值
   ```

2. ```java
   public Future<?> submit()	// 任务执行完成后有返回值
   ```

**线程池监控**

```java
public long getTaskCount()	//线程池已执行与未执行的任务总数
public long getCompletedTaskCount()	// 已完成的任务数
public int getPoolSize()	// 线程池当前的线程数
public int getActiveCount()		// 线程池中正在执行任务的线程数量
```



**线程池原理**

![image-20201006171809503](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201006171809503.png)



### 源码分析

**execute方法**

execute()方法执行流程如下：

![image-20201006171943542](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201006171943542.png)

该方法在ThreadPoolExecutor中实现

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        
		// clt记录着runState和workerCount
        int c = ctl.get();
        /*
        	private static int workerCountOf(int c)  { return c & CAPACITY; }
        	workCountOf方法取出低29位值，表示当前活动的任务数；
        	如果任务数小于corePoolSize，则新建一个线程放入线程池中；
        	并把任务添加到该线程中。
        */
        if (workerCountOf(c) < corePoolSize) {
           	// 向线程池中新添加一个线程并让该线程处理这个任务
            if (addWorker(command, true))
                return;
            // 如果添加失败，则重新获取ctl值
            c = ctl.get();
        }
        // 走在这儿就是  线程数>=核心线程数
        // 如果线程池是RUNNING状态，就把这个任务放到队列workQueue中。
        if (isRunning(c) && workQueue.offer(command)) {
            // 重新获取ctl值
            int recheck = ctl.get();
            // 再次判断线程池的运行状态，如果不是运行状态，由于之前已经把comand添加到workQueue中了，
            // 这是需要移除该command（任务）
            // 并通过handler使用的拒绝策略对该任务进行拒绝
            if (! isRunning(recheck) && remove(command))
                reject(command);
            /*
            	获取线程池中的有效线程数，如果数量是0，则执行addWorker方法
            	这里传入的参数表示：
            		1. 第一个参数null，表示在线程池中创建一个线程，但不去启动；
            		2. 第二个参数为false，将线程池的有限线程数量的上限设置为maximumPoolSize，添加线程时根据maximumPoolSize来判断；
           		如果判断workerCout大于0，则直接返回，在workQueue中新增的command会在将来的某个时刻被执行。
            */
            // 此时任务数为0，新建一个不绑定的任务的线程去处理队列中的任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        /*
        	如果执行到这里，有两种情况：
        		1. 线程池已经不是RUNNING状态；
        		2. 线程池是RUNNINF状态，但是workerCount >= corePoolSize并且workQueue已满。
        	这是，再次调用addWorker方法，但第二个参数传入false，将线程池的有限线程数量的上限设置为maximumPoolSize；
        	如果失败则拒绝该任务
        */
        // 此时workerCount >= corePoolSize并且workQueue已满。拒绝
        else if (!addWorker(command, false))
            reject(command);
    }
```



**addWorker()方法**

**addWorker()方法 用于在线程池中添加并启动工作线程**，有两个参数：

1. firstTask参数：用于指定新增的线程执行的第一个任务。
2. core参数：为true表示使用corePoolSize，为false表示使用maximumPoolSize

**注意注意！！！ 如果firstTask参数为null，那么我们只是新建了一个没有绑定任务的工作线程（Worker），该Worker就会从工作队列中取任务执行。**

**addWorker()用于添加线程并启动！！！！！！**

```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            /*
            	rs >= SHUTDOWN，表示此时线程池不在接收新任务
            	这里返回false有以下可能：
            		1. 线程池状态大于SHUTDOWN
            		2. 线程池状态为SHUTDOWN，但fistTask不为空，也就是线程池已经SHUTDOWN，拒绝添加新任务
            		3. 线程池状态为SHUTDOWN且firstTask为空，但是workQueue为空，即无任务需要执行,因为队列中没有任务了，不需要再添加线程了。
            		
            	注意注意！！！ 
            	如果线程池状态为SHUTDOWN且firstTask为空，但是workQueue不为空，此时队列中有任务需要添加线程去处理，当然要返回true
            */
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                /*
                	返回false有以下可能：
                		1. 任务数超过最大容量
                		2. 任务大于核心线程数量或最大线程数量
                */
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;	// CAS设置任务+1成功，直接跳过循环
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)	//线程池状态改变则重新循环判断
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
        
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
					/*
						持有锁后需要重新检查线程池的状态，防止ThreadTactory返回失败或者线程池在加锁之后被关闭
					*/
                    int rs = runStateOf(ctl.get());
                    /*
                    	如果rs是RUNNING状态或者
                    		rs是SHUTDOWN状态并且firstTask为null，线程池中添加线程
                    	因为在SHUTDOWN时不会添加新的任务但是会执行workQueue中的任务
                    */
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // workers是一个HashSet
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;	//记录着线程池中出现过的最大线程数量
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    // 启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```



### Worker类

​	线程池中的每一个线程被封装成一个Worker对象，ThreadPool维护的就是一组Worker对象。

```java
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }

```

Worker类是ThreadPoolExecutor的内部内，Worker类实现了Runnable，同时也扩展了AQS

并且我们可以看到Worker类实现了一个简单的不可重入互斥锁，工作线程在执行任务时，首先会进行加锁，如果主线程想要中断当前工作线程，需要先获取锁，否则无法中断。由此可知，只有等待从workQueue中获取任务（日奥用getTask期间）时才能中断。工作线程接受到中断信息，并不会立即就会停止，而是会检查workQueue是否为空，不为空则还是会继续获取任务执行，**只有队列为空才可以被中断。**

为什么Worker被设计为不可重入？

下面这些方法会发生中断工作线程：

- setCoolPoolSize()
- setMaximumPoolSize()
- setKeepAliveTime()
- allCoreThreadTimeOut()
- shutdown()
- tryTerminate()

如果锁可重入，调用以上方法配置和管理线程池时就可以再次获取锁，那么可能会导致调用这些方法过程中中断正在执行的工作线程。jdk不希望再调用想setCorePoolSize这样的线程池控制方法时重新获取锁。

此外我们从源码中可以知道：

```java
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
```

在Worker的构造函数中setState(-1) ,  是因为AQS中默认的state是0，如果刚刚创建了一个Worker对象，还没有执行任务时，这是就不应该被中断，在看看Worker实现的tryAcquire方法:

```java
        protected boolean tryAcquire(int unused) {
            // cas修改state，不可重入
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }
```



**runWorker方法**

在Worker类中的run方法调用了runWorker方法来执行任务，

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        // 获取第一个任务
        Runnable task = w.firstTask;
        w.firstTask = null;
        // 允许中断
        w.unlock(); 
        // 是否因为异常退出循环
        boolean completedAbruptly = true;
        try {
            // 如果task为空，则通过getTask来获取任务
            while (task != null || (task = getTask()) != null) {
                w.lock();
                if ((runStateAtLeast(ctl.get(), STOP) ||	
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

我们把runWorker方法中的那个if摘出来解释一下：

```java
if ((runStateAtLeast(ctl.get(), STOP) ||	// c >= s
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
```

我们在Thread类详解的那一篇中讲过中断的内容

- interrupted() 查询中断标识位，并清空中断标识
- isInterrupted()	查询中断标识位

这个if的理解：

​	如果线程池正在STOP的过程，我们之前说过线程池这个状态下中断当前正在执行的线程。

​	那么 

```java
(runStateAtLeast(ctl.get(), STOP)||(Thread.interrupted()&&runStateAtLeast(ctl.get(), STOP)))
```

```java
                   && !wt.isInterrupted())
```

```java
                    wt.interrupt();
```

如果当前线程池已经是STOP`(runStateAtLeast(ctl.get(), STOP)`，但是此时当前线程wt并并没有中断`!wt.isInterrupted())`, 那么我们肯定需要中断当前线程`wt.interrupt();`

如果当前线程池是RUNNING或这是SHURDOWN状态，那么我们要确保线程是非中断状态的，`Thread.interrupted()`保证了线程是非中断状态，此时还要重新检查线程池的状态，如果已经变成STOP状态了，但是此时当前线程wt并并没有中断`!wt.isInterrupted())`, 那么我们肯定需要中断当前线程`wt.interrupt();`



总结一下runWorker方法的执行过程：

1. while循环不断地通过getTask()方法获取任务；
2. getTask()方法从阻塞队列中取任务；
3. 如果线程池正在停止，那么要保证当前线程是中断状态，否则要保证当前线程不是中断状态；
4. 调用`task.run()`执行任务；
5. 如果task为空则跳出循环，执行processWorkerExit()方法；
6. runWorker方法执行完毕，也代表着Worker中的run方法执行完毕。这里的beforeExecute方法和affterExecute方法在ThreadPoolExecutor类中是空的，留给子类实现。

completedAbruptly变量表示在执行任务过程中是否出现了异常。



**getTask方法**

getTask方法是线程从阻塞队列中取任务，源码如下：

```java
    private Runnable getTask() {
        // timeout变量表示上次从阻塞队列中取任务时是否超时
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 如果线程池非RUNNING状态，并且是stop或者阻塞队列为空，则返回null
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            /*
            	timed用于判断是否需要进行超时控制。
            	allowCoreThreadTimeOut默认值为false，也就是核心线程不循序进行超时
            	wc > corePoolSize, 表示当前线程池中的线程数量大于核心线程数量；
            	timed = true，也就是对于超过核心线程的这些数量，需要进行超时控制
            */
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
			
            /*
            	wc > maximumPoolSize的情况是因为在此方法执行阶段同时执行了setMaximumPoolSize方法；
            	timed&&tineout 如果为true，表示当前操作需要进行超时控制；
            	接下来判断，如果有效线程数>1，或者阻塞队列为空，那么尝试将workerCount减1，如果减1失败，则开启新一轮循环重试
            	
            */
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                /*
                	根据timed来判断：
                	如果为true，则通过阻塞队列的poll方法进行超时控制，如果在keepAliveTime时间内没有获取到任务，则返回null
                	否则通过take方法，如果这是队列为空，则take方法会阻塞知道队列不为空。
                */
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                // 如果r == null 说明已经超时，timeout设为true
                timedOut = true;
            } catch (InterruptedException retry) {
                // 如果获取任务时当前线程发生了中断，则设置timeout为false并返回循环重试
                timedOut = false;
            }
        }
    }
```

总结：

 	这里最重要的地方就是第二个if判断，目的是控制线程池的有效线程数量。由我们上文分析的：

在执行execute方法时，如果线程池的数量超过了corePoolSize且小于maximumPoolSize，并且workQueue已满时，则可以增加工作线程，但这是如果超时没有获取到任务，也就是timeOut为true的情况，说明workQueue已经为空了，也说明了当前线程池中不需要这么多线程来执行任务了，可以把多余的corePoolSize数量的线程销毁，保持线程数量在corePoolSize即可。

​	那么问题来了？什么什么销毁？

当然是runWorker方法执行完之后，也就是Worker中的run方法执行完，由JVM自动回收。

getTask方法返回null时，在runWorker方法中跳出while循环，然后会执行processWorkExit方法。



**processWorkExit方法**

```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        /*
        	如果completedAbruptly为true，则说明线程执行时出现了异常，需要将workerCount减1；
        	如果线程执行是没有出现异常，说明getTask()方法中已经对workerCount进行了减1操作，这里就不用减了。
        */
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 统计完成的任务数
            completedTaskCount += w.completedTasks;
            // 从wokers中移除，也就表示着从线程池中移除了一个工作线程
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }
        // 根据线程池状态进行判断是否结束线程池
        tryTerminate();

        int c = ctl.get();
        /*
        	当线程池是RUNNING或SHUTDOWN状态时，如果woker是异常结束，那么会直接addWorker；
        	如果allCoreThreadTimeOut = true，并且等待队列不为空，至少保留一个woker；
        	如果allCoreThreadTimeOut = false，workerCount不少于corePoolSize，直接返回
        */
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
```

从execute方法开始，Worker使用ThreadFactory创建新的工作线程，runWorker通过getTask获取，然后执行任务，如果getTask返回为null，进入processWorkExit，这个线程结束

![image-20201007175628418](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201007175628418.png)



### Executor使用demo

```java
public class ExecutorDemo {
    // 获取cpu核数
    public static int nThreads = 2 * Runtime.getRuntime().availableProcessors();
    public static void main(String[] args) {
        ExecutorService pool = Executors.newFixedThreadPool(nThreads);
        AtomicInteger sum = new AtomicInteger(0);
        for (int i = 0; i < 100; i++) {
            pool.execute(()->{
                sum.getAndIncrement();
                System.out.println("====="+((ThreadPoolExecutor)pool).getActiveCount()+"  sum:"+sum.get());
            });
        }
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            System.out.println("***"+((ThreadPoolExecutor)pool).getActiveCount());
            pool.shutdown();
        }
        System.out.println("main线程");
    }
}
```

![image-20201007184835848](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201007184835848.png)



**有返回值的ThreadPoolExecutor**

​	求200万随机数数组的和

```java
public class HasRetExecutorDemo {
    private final static int NCPU = Runtime.getRuntime().availableProcessors();
    private final static int NUM = 1000;
    public static void main(String[] args) throws InterruptedException {
        int[] array = Utils.buildRandomIntArray(2000000);
        long result = 0;
        int numThreads = array.length / NUM > 0 ? array.length / NUM : 1;
        ExecutorService pool = Executors.newFixedThreadPool(NCPU);
        try {
            SumTask[] tasks = new SumTask[numThreads];
            Future<Long>[] sums = new Future[numThreads];
            for (int i = 0; i < numThreads; i++) {
                tasks[i] = new SumTask(array, (i * NUM), (i + 1) * NUM);
                sums[i] = pool.submit(tasks[i]);
            }
            for (int i = 0; i < numThreads; i++) {
                result += sums[i].get();
            }
        } catch(InterruptedException e){
                e.printStackTrace();
        } catch(ExecutionException e){
                e.printStackTrace();
        } finally {
            pool.shutdown();
        }
        System.out.println("sum =="+ result);
        Thread.sleep(3000);
        System.out.println("sum =="+ result);
        System.out.println("stream sum =="+ Arrays.stream(array).sum());
    }

    static class SumTask implements Callable<Long> {
        int lo;
        int hi;
        int[] array;
        public SumTask(int[] arr, int l, int h) {
            lo = l;
            hi = h;
            array = arr;
        }
        @Override
        public Long call() {
            long result = 0;
            for (int i = lo; i < hi; i++) {
                result += array[i];
            }
            return result;
        }
    }

}
```

![image-20201008085610263](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201008085610263.png)