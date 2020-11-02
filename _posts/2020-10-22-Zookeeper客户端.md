---
title: Zookeeper客户端
tags:

  - 分布式系统
---

### Zookeeper客户端

我这儿新建一个zookeeper源码的模块

写了个main方法后，发现不答应日志，解决办法：

将conf下面的`log4j.properties`复制到该模块的resoures目录下。

#### zookeeper原生

依赖：

```xml
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.6.1</version>
        </dependency>
```

代码：

```java
public class ZookeeperClientTest {
    public static void main(String[] args) throws Exception{
        // 默认的Watch,
        ZooKeeper client = new ZooKeeper("localhost:2181", 5000,
                new Watcher() {
            @Override
            // 连接服务端的时候执行
            public void process(WatchedEvent event) {
                System.out.println("连接的时候：---> " + event);
            }
        });

        // 数据发生了改变的时候
        client.getData("/alonge", new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (Event.EventType.NodeDataChanged.equals(event.getType())) {
                    System.out.println("数据发生了改变");
                }
            }
        }, new Stat());

        client.getData("/alonge", true, new Stat());
        System.in.read();
    }
}
```

当运行这个main方法的时候：

![image-20201031202924663](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031202924663.png)

当我们改变 /alonge 的数据的时候：即

```
[zk: 192.168.1.106:2181(CONNECTED) 1] set /alonge "joelove"$<2>
```

程序WatchedEvent.NodeDataChanged响应了：

![image-20201031203215702](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031203215702.png)

但是当我们继续改变 /alonge 这个znode节点的数据时，发现这个watch失效了，也就是这个是一次性的。

基于这种watch可以编写我们的发布者订阅模式。



getData方法：

![image-20201031204055196](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031204055196.png)

我们看到有一个返回 boolean 的watch，那么这个是什么意思呢？

我们看源码：

![image-20201031203815640](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031203815640.png)

可以看到如果为true，就给一个默认的watch（即连接的时候执行任务）；

如果为false，则不指定watch。



![image-20201031204338520](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031204338520.png)

可以看到，我们改变alonge的数据的时候，输出了我们new 一个Zk客服端是给定的默认watch。



同样只执行一次就会失效。



#### zkClient

zkClient是对原生zookeeper的封装。

依赖：

```xml
        <!-- https://mvnrepository.com/artifact/com.101tec/zkclient -->
        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.11</version>
        </dependency>
```

代码：

```java
public class ZkClientTest {
    public static void main(String[] args) throws Exception {
        ZkClient client = new ZkClient("localhost:2181",
                10000, 10000, new SerializableSerializer());
        // 创建持久化节点
        client.createPersistent("/joelovealonge", "joelove");
        // watch 即 监听事件改变
        client.subscribeDataChanges("/joelovealonge", new IZkDataListener() {
            @Override
            // 数据改变事件
            public void handleDataChange(String s, Object o) throws Exception {
                System.out.println("数据发生了改变");
            }

            @Override
            public void handleDataDeleted(String s) throws Exception {

            }
        });
        System.in.read();
    }
```

当我们改变 /joelovealonge 节点的数据时，

![image-20201031210255230](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031210255230.png)

发现报错，这个错误是因为序列化不同。

```java
public class ZkClientTest {
    public static void main(String[] args) throws Exception {
        ZkClient client = new ZkClient("localhost:2181",
                10000, 10000, new SerializableSerializer());
        // 创建持久化节点
        // client.createPersistent("/joelovealonge", "joelove");
        // watch 即 监听事件改变
        client.subscribeDataChanges("/joelovealonge", new IZkDataListener() {
            @Override
            // 数据改变事件
            public void handleDataChange(String s, Object o) throws Exception {
                System.out.println("数据发生了改变");
            }

            @Override
            public void handleDataDeleted(String s) throws Exception {

            }
        });
        Thread.sleep(2000);
        client.writeData("/joelovealonge", 1111);
        System.in.read();
    }
```

以这种方式写就没问题，2秒后打印了数据发生了改变。

序列化自行查找资料。

watch也是一次性的。



#### Curator客户端

流式的写法。

依赖：

```xml
<!-- https://mvnrepository.com/artifact/org.apache.curator/curator-framework -->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>5.1.0</version>
</dependency>

```

未完待续==> 因为新增了一个CuratorCache

