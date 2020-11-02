---
title: 搭建Zookeeper源码环境
tags:

  - 分布式系统
---



###  搭建Zookeeper源码环境

#### 下载编译

1. 去github上下载zookeeper。

   https://github.com/apache/zookeeper/

2. 根据github上说的使用maven编译：

   ![image-20201031160627943](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031160627943.png)

   需要java8，然后我的maven版本是3.5.2

   进入本地zookeeper项目中：

   按照github上说的使用命令：`mvn clean install` 进行编译。

   在这里我使用的是：

   ```
   mvn clean install -Dmaven.test.skip
   跳过测试类
   ```

   然后等一段时间，还挺快。

   ![image-20201031161012140](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031161012140.png)

   BUILD SUCCESS, 编译成功


#### 我遇到的坑

   第一：

   ![image-20201031161842225](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031161842225.png)

   意思就是maven插件清除zookeeper-jute jar包失败，找到项目中该模块的target文件，将里面的内容删除后重新编译。

   第二：

   ![image-20201031162057687](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031162057687.png)

   我去这个仓库 https://repository.apache.org/content/groups/snapshots/org/apache/zookeeper/zookeeper/  查看后如下：

   ![image-20201031162155351](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031162155351.png)

发现仓库没有提供3.6.3的依赖。   

因为我刚开始下载的3.6分支（具体是3.6.3），然后我重新下载3.6.1分支，然后重新编译成功。



#### idea打开并运行

新建一个zoo.conf 文件，内容如下：

![image-20201031165723807](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031165723807.png)

新建Run Configuration

![image-20201031165011109](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031165011109.png)

运行：

![image-20201031165749864](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031165749864.png)

无法输出日志，

解决办法：

```
-Dlog4j.configuration=file:E:\ideaworkspace\zookeeper-release-3.6.1\conf\log4j.properties
```

在Run configuration 中 配置vm options：

![image-20201031170746946](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031170746946.png)

启动成功：

![image-20201031170839694](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031170839694.png)

用客户端zkCli.sh 也可以连接到：

![image-20201031171048693](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201031171048693.png)