---
title: HystrixDashbord
tags:

  - SpringCloud
---

### HystrixDashbord

hystrix（单纯的hystrix）提供了对微服务调用状态的监控（信息），但是需要结合`spring-boot-actuator`模块一起使用。

在包含hystrix的项目中，引入依赖：

```xml
        <!--actuator  hystrix 提供了对于微服务调用状态的监控信息，但是需要结合actuator-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

此外还需要加入下面配置：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

因为springboot2.0.x以后的actuator暴露了info和health 2个端点，这里我们把所有端点开放。

![image-20201028181217418](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201028181217418.png)

这些密密麻麻的，就是我们的微服务监控的信息，但是，这种json格式的字符串，难免会让人不好阅读，所以我们的主角`HystrixDashbord`登场了：



###  什么是HystrixDashbord/如何使用？

Dashbord翻译一下的意思是仪表盘，顾名思义，hystrix监控信息的仪表盘，那如何使用呢？

加入依赖：

```xml
        <!--dashboard -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
        </dependency>
```

在启动类上加上注解@EnableHystrixDashboard，（这里我就在User项目中集成）

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
@EnableHystrix
@EnableHystrixDashboard
public class AppUser {
    public static void main(String[] args) {
        SpringApplication.run(AppUser.class);
    }
}
```

![image-20201028181927316](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201028181927316.png)

在中间这个输入框中，填入需要监控的微服务地址也就是 /actuator/hystrix.stream ,点击按钮：

![image-20201028183334868](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201028183334868.png)

发现连接不上，我们查后台发现：

```java
2020-10-28 18:33:16.269  WARN 10136 --- [nio-8080-exec-5] ashboardConfiguration$ProxyStreamServlet : Origin parameter: http://localhost:8080/actuator/hystrix.stream is not in the allowed list of proxy host names.  If it should be allowed add it to hystrix.dashboard.proxyStreamAllowList.

```

在配置文件中加入：

```yaml
hystrix:
  dashboard:
    proxy-stream-allow-list: "localhost"
```

然后就可以了

![image-20201028183157990](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201028183157990.png)

#### Hystrix仪表盘解释

![image-20201028183712381](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image-20201028183712381.png)

实心圆有两种含义：

1. 颜色代表实例的健康程度，绿绿更健康，越绿越健康。
2. 大小表示请求流量的变化，流量越大该实心圆越大。所以通过该实心圆的展示，就可以在大量的实例中快速的发现故障实例和高压力实例。

曲线：用来记录2分钟内流量的相对变化，可以通过它来观察流量的上升和下降趋势。

其他的看上图。

