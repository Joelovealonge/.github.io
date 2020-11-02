---
title: Zookeeper详细功能
tags:

  - 分布式系统
---

### Zookeeper详细功能

##### 节点类型（znode）

1. 持久节点

   指节点创建后，就一直存在，直到有删除操作来主动清除这个节点。

2. 临时节点

   临时节点的生命周期和和客户端会话绑定。也就是说，如果客户端会话失效，那么这个节点就会被自动清除掉。注意，这里提到的会话失效，而非断开连接。另外，在临时节点下面不能创建子节点。

3. 持久顺序节点

   这类节点基本特性和持久节点是一致的，额外的特性是，在zk中，每个父节点会为它的第一级子节点维护一份时序，会记录每个子节点创建的先后顺序。基于这个特性，在创建子节点的时候，就可以设置这个属性，那么在创建节点过程中，zk会自动为给定节点加上一个数字后缀，作为新的节点名。这个数字后缀的范围是整数的最大值。

4. 临时顺序节点

   类似临时节点和顺序节点。

**zk默认对每个节点的最大数据量有一个上限是1M**



##### Stat

zk命名空间中的每一个znod都有一个与之关联的stat结构，类似于linux文件系统中文件的stat结构。znode的state结构中的字段显示如下，各自的含义如下：

| 字段           | 含义                                                         |
| -------------- | ------------------------------------------------------------ |
| cZxid          | 创建znode的事务ID                                            |
| mZxid          | 最后修改znode的事务ID                                        |
| pZxid          | 最后修改添加或删除子节点的事务ID                             |
| ctime          | 表示从1970-01-01T:00:00:00Z开始以毫秒为单位的znode创建时间   |
| mtime          | 表示从1970-01-01T:00:00:00Z开始以毫秒为单位的znode最后修改时间 |
| dataVersion    | 表示对该znode的数据所做的更改次数                            |
| cversion       | 表示对此znode的子节点进行更改的次数                          |
| aclVersion     | 表示对此znode的ACL进行更改的次数                             |
| ephemeralOwner | 如果znode是ephemeral类型的节点，则这是znode所有者的session ID。如果znode不是ephemeral节点，则该字段设置为零 |
| dataLength     | 这是znode数据字段的长度                                      |
| numChildren    | 这表示znode的子节点的数量。                                  |



zxid-后面篇幅说：

​	类似于RDBMS中的事务ID，用于标识一次更新操作Proposal ID。为了保证顺序性，该Zxid必须单调递增。因此zk使用一个64位的数来表示，高32位是Leader的epoch，从1开始，每次选出新的Leader，epoch加一；低32位为该epoch内的序号，没次epoch变化，都将低32位的序号重置。这样保证了zkid的全局递增性。



##### Watch

一个zk的节点可以被监控，包括这个目录中存储的数据的修改，子节点目录的变化。一旦变化可以通过设置监控的客户端，这个功能是在看对于应用最重要的特性，通过这个特性可以实现的功能包括配置的集中管理、集群管理、分布式锁等。

Watch机制官方说明：一个Watch事件是一个一次性的触发器。当被设置了Watch的数据发生了改变的时候，则服务器将这个改变发送给设置了Watch的客户端，以便通知它们。

可以注册Watcher的方法：`getData、exists、getChildren`。

可以出发Watcher的方法：`create、delete、setData`。连接断开的情况下触发的Watcher会丢失。

一个watcher实例是一个回调函数，被调用一个后就会被移除了。如果还需要关注数据的变化，需要再次注册watcher。

New Zookeeper时注册的watcher叫default watcher，它不是一次性的，只对client的连接状态变化作出反应



|                        | event For “/path”     | event For "/path/child" |
| ---------------------- | --------------------- | ----------------------- |
| create("/path")        | EventType.NodeCreated | 无                      |
| delete("/path")        | EventType.NodeDeleted | 无                      |
| setData("/path")       | EventType.NodeDataChanged | 无                      |
| create("/path/child")  | EventType.NodeChildrenChanged(getChild)                   | EventType.NodeCreated |
| delete("/path/child")  |     EventType.NodeChildrenChanged(getChild)                    | EventType.NodeDeleted |
| setData("/path/child") |          无             | EventType.NodeDataChanged |



| event For “/path”             | Default Watcher | exists("/path") | getData("/path") | getChildren("/path") |
| ----------------------------- | --------------- | --------------- | ---------------- | -------------------- |
| EventType.Node                | ✔               | ✔               | ✔                | ✔                    |
| EventType.NodeCreated         |                 | ✔               | ✔                |                      |
| EventType.NodeDeleted         |                 | ✔               | ✔                |                      |
| EventType.NodeDataChanged     |                 | ✔               | ✔                |                      |
| EventType.NodeChildrenChanged |                 |                 |                  | ✔                    |



**exists和getData设置数据监视，而getChildren设置子节点监视**



#### 常用命令



##### 创建节点(znode)

用给定的路径创建一个节点。flag参数指定节点是临时、持久还是顺序的。

默认情况下，所有节点都是持久的。

当会话过期或客户端断开连接时，临时节点（flag：-e）将被自动删除

顺序节点保证节点路径将是唯一的。zk将向节点路径填充1位序列化，李儒，节点路径/myapp将转换为/myapp0000000001, 下一个序列化将为/myapp0000000002。

如果没有指定flag。则节点认为是持久的。

语法：

```
create [-e/-s] /path date
# 解释
# -s 创建顺序节点； -e 创建临时节点
```

示例：

创建持久节点

```
create /firstnode firstdata
```

![image-20201031130813230](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031130813230.png)

创建顺序节点：

```
create -s /seqnode seqnodedata
```

![image-20201031131026811](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031131026811.png)

创建临时节点：

```
create -e /ephemeralnode ephemeraldata
```

![image-20201031131210379](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031131210379.png)



##### 获取数据

它返回节点的关联数据和指定节点的元数据。

语法：

```
get /path
```

示例：

```
get /firstnode
```

![image-20201031131507312](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031131507312.png)



##### 设置数据

设置指定znode的数据。完成此操作后，你可以使用getCLI命令来检查数据。

语法：

```
set /path data
```

示例：

```
set /firstnode firstnodedataupdate
```

![image-20201031132635566](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031132635566.png)



##### 创建子节点

创建子节点类似于创建新的znode。唯一的区别是，子node的路径也将具有父路径。

语法：

```
create /parent/path/subnode/path data
```

示例：

```
create /firstnode/childnode childnodedata
```

![image-20201031133102838](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031133102838.png)



##### 列出子节点

此命令用于列出znode的子项

语法：

```
ls /path
```

实例：

```
ls /firstnode
```

![image-20201031133444845](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031133444845.png)



##### 检查状态

状态描述指定的znode的元数据。它包含时间戳，版本号，ACL，数据长度和子znode等。

语法：

```
stat /path
```

示例：

```
stat /firstnode
```

![image-20201031133703457](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031133703457.png)



##### 移除znode

deleteall移除指定的znode并递归其所有子节点。只有在znode可用的情况下才会发生。

语法：

```
deleteall /path
```

示例：

```
deleteall /firstnode
```

![image-20201031134456054](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031134456054.png)

delete 只适用于没有子节点的znode



##### ACL

zk做为分布式架构中的重要中间件，通常会在上面以节点的方式存储一些关键信息，默认情况下，所有应用都可以读写任何节点，在复杂的应用中，这不太安全，zk通过ACL机制来解决权限问题。

- zk的权限控制是基于每个znode节点的，需要对每个节点设置权限
- 每个znode支持设置多种权限控制方案和多个权限
- 子节点不会继承父节点的权限，客户端无权访问某节点，但可能可以访问它的子节点



ACL权限控制，使用： `schema:id:permission` 来表示，主要涵盖3个方面：

- 权限模式（schema）：鉴权的策略
- 授权对象（ID）
- 权限（permission）



###### schema

- world：只有一个用户：anyone，代表所有人（默认）
- ip：使用ip地址认证
- auth：使用已添加认证的用户认证
- digest：使用“用户名：密码”方式认证



###### ID

授权对象ID是指，权限赋予的用户或者一个实例，例如：IP地址或机器。

授权模式与授权ID之间的关系：

- world：只有一个id，即anyone
- ip：通常是一个ip地址或地址段，比如192.168.0.110或192.168.0.1/24
- auth：用户名
- digest：自定义：通常是“username:BASE64(SHA-1(username:password))”



###### 权限

- create，简写c，可以创建子节点
- delete，简写d，可以删除子节点（仅下一级节点），注意不是本节点
- read，简写r，可以读取节点数据及显示子节点列表
- write，简写w，可设置节点数据
- admin，简写a，可以设置节点访问列表



###### 查看ACL

```
getAcl /zookeeper
```

![image-20201031140151791](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031140151791.png)

默认创建的节点的权限是最开放的，所有都可以增删改查管理。



##### 设置ACL

设置节点对所有人都有写和管理的权限

```
setAcl /testaclnode world:anyone:wa
```

读取时会提示

![image-20201031140915793](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031140915793.png)

先添加用户：

```
addauth digest zhangsan:12345
```

在设置权限，这个节点只有zhangsan这个用户拥有所有权限

```
setAcl /testaclnode auth:zhangsan:12345:rdwca
```

在本个连接中，可以get /testaclnode 节点的数据，

但是我新开了一个客户端连接，拿不到/testaclnode的数据。



