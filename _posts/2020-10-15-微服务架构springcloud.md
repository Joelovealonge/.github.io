---
title: 初识微服务架构
tags:

  - SpringCloud
---

### 什么是微服务？

微服务就是吧原本臃肿的一个项目的所有模块拆分开来做到相互没有联系，甚至可以不使用同一个数据库。把模块单独分成一个服务(项目)。

### 微服务和分布式的区别

所谓分布式，就是将偌大的系统划分为多个模块部署到不同机器上（因为一个机器可能承受不了这么大的压力），各个模块通过接口进行数据交互，其实分布式也是一种微服务。

它们本质的区别主要体现在目标上：就是你这样架构项目要做的事情

- 分布式的目标：高并发，一台机器承载不了。
- 微服务的目标：让各个模块拆分开来，不会相互影响，模块的升级不会影响到其他模块等等。

### Spring-Cloud

微服务是一种项目架构方式，spring-cloud是微服务的一种技术实现。

spring-cloud可以解决微服务架构中的各种问题：负载均衡，服务注册与发现，服务调用，路由等等。

#### Spring-Cloud项目的搭建

spring-loud是基于spring-boot项目来的。注意spring-cloud与spring-boot的版本对应关系。

在这里我用spring-cloud的最新版本Hoxton.SR8。

![image-20201026190316620](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201026190316620.png)

从官网可以看到 https://docs.spring.io/spring-cloud/docs/Hoxton.SR8/reference/html/  该版本需要Boot Version: **2.3.3.RELEASE**。

新建一个spring-cloud-study父项目。父项目依赖

```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.3.RELEASE</version>
    </parent>

    <!-- spring cloud版本管理-->
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Hoxton.SR8</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

新建一个模块first，依赖

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
```

![image-20201026193943271](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201026193943271.png)

好了，我们的项目就可以跑起来了。

下面介绍Spring-Cloud的核心组件。

### Spring-Cloud 组件

#### eureka

eureka是Netflix的子模块之一，也是一个核心的模块。

eureka有两个组件：

1. Eureka Server（一个独立的项目）

   这个是用于定位服务以实现中间层的负载均衡和故障转移

2. Eureka Client（我们的微服务）

   用于与Eureka Server进行交互，可以使得交互变得非常简单，只需通过服务标识符即可拿到服务。

Spring-Cloud 封装了Netflix公司的**Eureka模块来实现服务注册与发现**（可以对比Zookeeper）。



Eureka采用C-S架构。Eureka Server作为服务注册功能的服务器，它是注册中心。

而系统中的其他服务，使用Eureka的客户端连接到Eureka Server并维持心跳。这样系统的维护人员可以通过Eureka Server来监控系统中各个微服务是否正常运行。spring cloud的一些其他模块（比如Zuul）就可以通过Eureka Server来发现系统中的其他微服务，并执行相关的逻辑。

![image-20201026195311016](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201026195311016.png)

我们来实际操作一下：

新建一个 eureka 模块：

pom依赖

```xml
        <!-- eureka server -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
```

application.yml 配置：

```yml
server:
  port: 3000
eureka:
  server:
    enable-self-preservation: false #关闭自我保护机制
    eviction-interval-timer-in-ms: 4000 #设置清理时间（单位毫秒 默认60*1000）
  instance:
    hostname: localhost

  client:
    register-with-eureka: false #不把自己作为一个客户端注册到自己身上
    fetch-registry: false #不需要从服务端获取注册信息（因为在这里自己就是服务端，而且已经禁用自己注册了）
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka

```

![image-20201026202405561](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201026202405561.png)

![image-20201026202455455](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201026202455455.png)

可以看到我们的注册中心服务端已经搭建好了。

接下来把我们的first服务注册到Eureka上。

加入依赖

```xml
        <!--Eureka 客户端 依赖-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
```

application.yml配置

```yaml
server:
  port: 8080
eureka:
  client:
    service-url:
      defaultZone: http://localhost:3000/eureka/  #eureka服务端提供的注册地址，参考服务端配置的这个路径
  instance:
    instance-id: first-1 #此实例注册到eureka服务端的唯一实例ID
    prefer-ip-address: true #是否显示IP地址
    lease-renewal-interval-in-seconds: 10 #eureka客户端需要多长时间发送心跳给eureka服务器，表名它仍然活着，默认为30秒（与下面配置的单位都是秒）
spring:
  application:
    name: first-server  #此实例注册到eureka服务端的name

```

![image-20201026203943071](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201026203943071.png)

![image-20201026204020894](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201026204020894.png)

可以看到我们的first-server服务已经被注册成功了。

