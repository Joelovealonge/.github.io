### ScheduledThreadPoolExecutor

定时线程池类的结构图

![image-20201008101034159](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201008101034159.png)

它用来**处理延时或定时任务**。

![image-20201008101147734](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201008101147734.png)

它接受ScheduleFuture任务，是线程池调度任务的最小单位，有三种提交任务的方式：

1. schedule
2. scheduleAtFixedRate
3. scheduleWithFixedDelay

它采用DelayQueue存储等待的任务

1. Delay内部封装了一个PriorityQueue，它会根据time的先后时间排序，若time相同则根据sequenceNumber排序；
2. DelayQueue也是一个无界队列；



### 定时类线程池Demo

```java
public class ScheduledThreadPoolDemo {
    public static void main(String[] args) {
        ScheduledExecutorService pool = Executors.newScheduledThreadPool(1);
        try {
            // 延期执行 只执行一次
            pool.schedule(() -> System.out.println("延迟执行"), 1, TimeUnit.SECONDS);

            // 这个执行周期是固定的，不管任务执行多长时间，每过3秒就产生一个新的任务
            pool.scheduleAtFixedRate(() -> System.out.println("重复执行"),1, 3, TimeUnit.SECONDS);

            // 任务执行完成后延期3秒，产生新的任务
            pool.scheduleWithFixedDelay(() -> System.out.println("重复执行delay"),1, 3, TimeUnit.SECONDS);
        } finally {
            pool.shutdown();
        }
    }
}
```

