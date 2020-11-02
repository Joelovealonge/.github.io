---
title: 分布式系统介绍
tags:

  - 分布式系统
---

### Zookeeper快速入门

Apache Zookeeper是Apache软件基金会的一个软件项目，他为大型分布式计算提供开源的分布式配置服务、同步服务和命名注册。Zookeeper曾经是Hadoop的一个子项目，然是现在是一个独立的顶级项目。

#### 下载与安装

下载地址：https://zookeeper.apache.org/

下载解压我就不说了，我们来直接配置：

进入`conf`文件夹，找到 `zoo_sample.cfg`, 直接复制一份`zoo.cfg`

![image-20201030202625086](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201030202625086.png)

我的配置：

![image-20201030204948659](C:\Users\wyl\AppData\Roaming\Typora\typora-user-images\image-20201030204948659.png)

#### 配置文件解释：

- tickTime：zk服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个tickTime时间会发送一个心跳。tickTime以毫秒为单位。该参数用来定义心跳的时间间隔，zk的客户端和服务端之间也有和web开发里类似的session概念，而zk里最小的session过期时间就是tickTime的两倍。
- initLimit：follower在启动过程中，会从Leader同步所有最新数据，然后确定自己能够对外服务的其实状态。Leader允许follower在initLimit时间内完成这个工作。通常情况下，我们不用太在意这个参数的设置。如果zk集群的数据量确实很大了，follower在启动的时候，从leader上同步数据的时间也会相应边长，因此此种情况下，有必要适当调大这个参数了。默认为10
- syncLimit：在运行过程中，Leader负责与zk集群中所有机器进行通信，例如通过一些心跳检测机制，来检测机器的存活状态。如果leader发出心跳包在syncLimit之后，还没有从follower哪里收到响应，那么认为这个follower已经不在线了。注意：不要把这个藏书设置的过大，否则可能会掩盖一些问题。
- dataDir：存储快照文件snapshot的目录。默认情况下，事务日志也会存储在这里。建议同时配置参数dataLogDir，事务日志的写性能直接影响zk性能。
- clientPort：客户端连接服务器的端口
- maxClientCnxns
- autopurge.snapRetainCount
- autopurge.purgeInterval



#### 启动服务端

首先将 `zkServer.sh` 和 `zkCli.sh` 加入到环境变量

```
vi etc/profile
在该文件中加入 export PATH=$PATH:/root/zookeeper/zookeeper-3.6.2/bin
source /etc/profile
```

启动服务：

```
zkServer.sh start
```

![image-20201030212729082](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201030212729082.png)

停止服务：

```
zkServer.sh stop
```

查看服务状态：

```
zkServer.sh status
```



#### 使用客户端

```
zkCli.sh
```

![image-20201030213029193](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201030213029193.png)



### 集群环境搭建

单机环境下模拟集群环境，不同端口模拟不同主机。

1. 复制配置文件

   ```
   [root@localhost conf]# cp zoo.cfg zoo1.cfg
   [root@localhost conf]# cp zoo.cfg zoo2.cfg
   [root@localhost conf]# cp zoo.cfg zoo3.cfg
   ```

2. 创建数据以及日志文件目录

   ```
   [root@localhost conf]# mkdir -p /zookeeper/data_1
   [root@localhost conf]# mkdir -p /zookeeper/data_2
   [root@localhost conf]# mkdir -p /zookeeper/data_3
   ```

3. 创建myid文件

   ```
   [root@localhost conf]# echo 1 > /zookeeper/data_1/myid
   [root@localhost conf]# echo 2 > /zookeeper/data_2/myid
   [root@localhost conf]# echo 3 > /zookeeper/data_3/myid
   ```

4. 修改配置文件

   zoo1.cfg

   ```
   tickTime=2000
   initLimit=10
   syncLimit=5
   dataDir=/zookeeper/data_1
   dataLogDir=/zookeeper/logs_1
   # 与客户端的通信端口
   clientPort=2181
   # server.x中的myid中的一致，第一个端口是Leader和learner（包括follower和observer）节点之间的
   同步，第二个端口用于选举过程中的投票通信
   server.1=localhost:2887:3887
   server.2=localhost:2888:3888
   server.3=localhost:2889:3889
   ```

   zoo2.cfg

   ```
   tickTime=2000
   initLimit=10
   syncLimit=5
   dataDir=/zookeeper/data_2
   dataLogDir=/zookeeper/logs_2
   # 与客户端的通信端口
   clientPort=2182
   # server.x中的myid中的一致，第一个端口是Leader和learner（包括follower和observer）节点之间的
   同步，第二个端口用于选举过程中的投票通信
   server.1=localhost:2887:3887
   server.2=localhost:2888:3888
   server.3=localhost:2889:3889
   ```

   zoo3.cfg

   ```
   tickTime=2000
   initLimit=10
   syncLimit=5
   dataDir=/zookeeper/data_3
   dataLogDir=/zookeeper/logs_3
   # 与客户端的通信端口
   clientPort=2183
   # server.x中的myid中的一致，第一个端口是Leader和learner（包括follower和observer）节点之间的
   同步，第二个端口用于选举过程中的投票通信
   server.1=localhost:2887:3887
   server.2=localhost:2888:3888
   server.3=localhost:2889:3889
   ```

5. 启动集群

   ```
   zkServer.sh start 对应的配置文件
   ```

   ![image-20201031100635473](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031100635473.png)

6. 验证

   ```
   zkServer.sh status 对应的配置文件
   ```

   ![image-20201031100837971](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031100837971.png)

   可以看到：一个Leader，两个Follower

7. 命令行连接zk集群

   ```
   zkCli.sh -server localhost:2181,localhost:2182,localhost:2183
   ```

   ![image-20201031101200089](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031101200089.png)

   也可以连接单个zk。



### 集群中的角色

- 领导者（Leader）

  负责进行投票的发起和决议，最终更新状态。

- 跟随者（Follower）

  用于接收客户请求并返回客户结果。参与Leader发起的投票。

- 观察者（Observer）

  可以接收客户端连接，将写请求转发给Leader节点，但是Observer不参加投票过程，只是同步Leader的状态。Observer为系统扩展提供了一种方法。

- 学习者（Learner）

  和Leader进行状态同步的节点统称为Learner，上述Follower和Observer都是Learner。



#### 为什么要有Observer

zk服务中的每个server可服务于多个client，并且client可连接到zk服务中的任一台server来提交请求。若是读请求，则每台server的本地副本数据库直接响应。若是改变server转态的写请求，需要通过一致性协议来处理，这个协议就是Zab协议。

简单来说，Zab协议规定：

​	来自Client的所有写请求，都要转发给ZK服务中唯一的Server----Leader，由Leader根据该请求发起一个Proposal。然后，其他的Server对该proposal进行投票。之后，leader对投票进行收集，当投票数量超过半时Leader会向所有的Server发送一个通知消息。最后Client所连接的Server收到该消息时，会把该操作操作更新到内存中并对Client的写请求作出回应。

Zk服务器在上述协议中实际扮演了两个只能，一方面从客户端接收连接与操作请求，另一方面对操作结果进行投票。这两个职能在zk集群扩展的时候彼此制约。

例如：当我们希望增加zk服务中的client数量的时候，那么我们需要增加server的数量，来支持这么多的客户端。然后从Zab协议对写请求的处理中我们可以发现，增加服务器的数量，则增加了对协议中投票过程的压力。**因为leader结点必须等待集群中过半server响应投票。**，于是结点的增加使部分计算机运行较慢，从而拖慢了整个投票过程。

即随着zk集群变大，写操作的吞吐量会下降。

随意引入不参与投票的服务器，称为Observer。Observer可以接收客户端的连接，并将写请求转发给Leader节点，但是，Leader节点不会要求Observer参与投票。

所以我们可以增加Observer几点，而无需太担心影响到写吞吐量。



##### Zookeeper中的CAP

zk至少满足了CP，牺牲了可用性，比如现在集群中的有Leader和Follower两种角色，那么当其中任意一台服务器挂掉了，都需要重新进行选举，在选举过程中，集群是不可用的，这就是牺牲了可用性。

但是，如果集群中有Leader、Follower、Observer三种角色，那么挂掉的是Observer，那么对于集群来说并没有影响，集群还是可以用的，只是Observer节点的数据不同了，从这个角度考虑，Zk又是牺牲了一致性，满足了AP



#### Zookeeper能做什么？

##### 统一命名服务

命名服务也是分布式系统中常见的一类场景。

在分布式系统中，通过命名服务，客户端应用能够根据指定名字来获取资源或服务器的地址，提供者等信息等。被命名的实体可以是集群中的及其，提供的服务地址，远程对象等等----这些我们都可以统称他们为名字（Name）。其中较为常见的就是一些分布式服务框架中的服务地址列表。通过调用ZK提供的创建结点的API，能够很容易创建出一个全局唯一的path，这个path就可以作为一个名称。



##### 配置中心

配置的管理在分布式应用场景中很常见，例如同一个应用系统需要多台PC server运行，但是他们运行的应用系统的某些配置项是相同的，如果要修改这些相同的配置项，那么久必须同时更改每台运行这个应用系统的PC Server，这样非常麻烦而且容易出错。

像这样的配置信息完全可以交给Zookeeper来管理，将配置信息保存在zk的某个目录节点中，然后将所有需要修改的应用及其监控配置信息的状态，一旦配置信息发生改变，每台应用及其就会收到zk的通知，然后从zk获取新的配置信息应用到系统中。



##### 集群管理和Master选举

集群机器监控：这通常用于那种对集群中及其状态，及其在线率有较高要求的场景，能够快速对集群中及其变化做出响应。这样的场景中，往往有一个监控系统，实时监测集群机器是否存活。

过去的做法通常是：监控系统通过某种手段（比如ping）定时检测每个及其，或者每个及其自己定时想监控系统汇报“我还活着“。

利用zk的两个特性，就可以实时另一种及其集群机器存活性监控系统：

1. 客户端在节点x上注册一个Watcher，那么如果x的子节点变化了，会通知该客户端。
2. 创建ephemeral类型的节点，一旦客户端和服务器的会话结束或过期，那么该节点就会消失。

例如：监控系统在 /clusterServers 节点上注册一个Watcher，以后没动态加及其，那么就往  /clusterServers  下创建一个EPHEMERAL 类型的节点： /clusterServers(hostname)。这样监控系统就能够实时知道及其的增减情况，至于后续处理就是监控系统的业务了。



在分布式环境中，相同的业务应用分布在不同的机器上，有一些业务逻辑（例如一些耗时的计算，网络I/O处理），往往只需要让整个集群中的某一台及其进行执行，其余机器可以共享这个结果，这样可以大大减少重复劳动，提高性能，于是master选举便是这种场景下。



##### 分布式锁

分布式锁，这个主要得益于zk为我们保证了数据的强一致性。锁服务可以分为两类，一个是保持独占，另一个是控制时序。



所谓保持独占，就是所有试图来获取这个锁的客户端，最终只有一个成功获得这把锁。通常的做法是把zk上的一个znode看做是一把锁，通过create znode的方式来实现。所有客户端都去创建 /distribute_lock 节点，最终成功创建的哪一个客户端也即拥有了这把锁。

控制时序，就是所有试图来获取这个锁的客户端，最终都是会被安排执行，只是有个全局时序了。做法和上面类似，只是这里 /distribute_lock 已经预先存在，客户端在它下面创建临时有序节点（这个可以通过节点的属性控制：CreateMode.EPHEMERAL_SEQUENTIAL来指定）。zk的父节点（/distribute_lock）维持一份sequence，保证子节点创建的时序性，从而也形成了每个客户端的全局时序。