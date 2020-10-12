---
title: springIOc
tags:

  - Spring应用
---

### spring IOC

IOC是控制反转，是面向对面编程中的一种设计原则，可以用来降低计算机之间的耦合性。其中最常见的方式叫做**依赖注入(DI)**，还有一种方式叫“依赖查找”。



Dependency Injection 依赖注入：

​	关于什么是依赖

​	关于注入和查找



为什么要使用Spring IOC

​	面向抽象编程会产生类的依赖，当然如果你够强大可以自己写一个管理的容器，但是既然spring以及实现了，并且spring如此优秀，我们仅仅需要学习spring框架便可。

当我们有了一个管理对象的容器之后，类的产生过程也交给了容器，至于我们自己的app则可以不需要去关系这些对象的产生了。



### Spring实现IOC的思路和方法

**spring实现IOC的思路是提供一些配置信息用来描述类之间的依赖关系，然后由容器去解析这些配置信息，继而维护对象之间的依赖关系。**

前提是对象之间的依赖必须在类中定义好，比如A.class中有一个B.class的属性，那么就是A依赖了B。既然我们在类中已经定义了他们之间的依赖关系那么为什么还需要在配置文件中去描述和定义呢？



Spring实现IOC的思路大致可以拆分成3点：

1. 应用程序中提供类，提供依赖关系（属性或构造方法）
2. 把需要交给容器管理的对象通过配置信息告诉容器（xml、annotation、javaconfig）
3. 把各个类之间的依赖关系通过配置告诉容器



配置这些信息的方法由三种：**xml，annotation、javaconfig**

维护的过程叫做自动注入，自动注入的方法有两种：**构造方法、setter**

自动注入的值可以是对象，数组，map，list个常量比如字符串整型等



Spring编程的风格

schemal-based  --------xml

annotation-based --------annotation

java-based --------java Configuration



### 注入的两种方法：

```
spring注入详细配置（字符串、数组等）参考文档：
https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-properties-detailed
```

**Constructor-based Dependency Injection**

```
构造方法注入参考文档：
https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-constructor-injection
```

例子：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    
    <bean id="dao" class="com.alonge.dao.IndexDaoImpl">
    </bean>
    <!-- 构造函数注入方式-->
    <bean id="indexService" class="com.alonge.service.IndexService">
        <constructor-arg ref="dao"/>
    </bean>
</beans>
```





**Setter-based Dependency Injection**

```
setter参考文档：
https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-setter-injection
```

例子：

```java
public class IndexService {
    private IndexDao indexDao;

    public void service(){
        indexDao.test();
    }

    public void setIndexDao(IndexDao indexDao) {
        this.indexDao = indexDao;
    }
}
```

```java
public class IndexDaoImpl implements IndexDao {
    @Override
    public void test() {
        System.out.println("test, I am a indexDao");
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <!-- setter 方式注入-->
    <bean id="dao" class="com.alonge.dao.IndexDaoImpl">
    </bean>
    <bean id="indexService" class="com.alonge.service.IndexService">
        <!--name属性的值（即indexDao） 绑定的是set方法，也就是setIndexDao，去掉set后首字母小写-->
        <property name="indexDao" ref="dao"></property>
    </bean>
</beans>
```

```java
public class Test {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring.xml");
        IndexService indexService = (IndexService) context.getBean("indexService");
        indexService.service();
    }
}
```

xml p和c命名空间

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="dao" class="com.alonge.dao.IndexDaoImpl">
    </bean>
    <!-- 构造函数注入方式-->
    <bean id="indexService" class="com.alonge.service.IndexService" c:indexDao-ref="dao">
        <!--<constructor-arg ref="dao"/>-->
    </bean>
    <!-- setter 方式注入-->
   <!-- <bean id="indexService" class="com.alonge.service.IndexService" p:str="dads " p:indexDao-ref="dao"/>-->
        <!--name属性的值（即indexDao） 绑定的是set方法，也就是setIndexDao，去掉set后首字母小写-->
        <!--<property name="indexDao" ref="dao"></property>-->
</beans>
```



通过注解声明一个对象，然后通过xml配置依赖关系，两者可以组合使用。

```java
@Component("dao")
public class IndexDaoImpl implements IndexDao {
    @Override
    public void test() {
        System.out.println("test, I am a indexDao");
    }
}
```

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">
    <!-- 开启注解编程并扫描-->
    <context:component-scan base-package="com"/>
    <!-- 构造函数注入方式-->
    <bean id="indexService" class="com.alonge.service.IndexService" c:indexDao-ref="dao">
        <!--<constructor-arg ref="dao"/>-->
    </bean>
    <!-- setter 方式注入-->
   <!-- <bean id="indexService" class="com.alonge.service.IndexService" p:str="dads " p:indexDao-ref="dao"/>-->
        <!--name属性的值（即indexDao） 绑定的是set方法，也就是setIndexDao，去掉set后首字母小写-->
        <!--<property name="indexDao" ref="dao"></property>-->
</beans>
```



**注解三者混用：**

```java
@Component("dao")
public class IndexDaoImpl implements IndexDao {
    @Override
    public void test() {
        System.out.println("test, I am a indexDao");
    }
}
```

```java
@Configuration
@ComponentScan("com")
@ImportResource("classpath:spring.xml")
public class AppConfig {
}
```

```java
public class Test {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        IndexService indexService = (IndexService) context.getBean("indexService");
        indexService.service();
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">
    
    <!-- 构造函数注入方式-->
    <bean id="indexService" class="com.alonge.service.IndexService" c:indexDao-ref="dao">
    </bean>
</beans>
```



官网上一句重要的话：

无论是哪一种方式xml、annotation、javaconfig。These sources are then converted internally into instances of `BeanDefinition` and used to load an entire Spring IoC container instance.



### 自动装配

​	上面说过，IOC的注入有两个地方需要提供依赖关系：一是类的定义中，二是在spring的配置中需要去描述。

​	自动转配则把第二个取消了，即我们仅仅需要在类的定义中提供依赖，继而把对象交给容器管理即可完成注入。

​	在实际开发中，描述类之间的依赖关系通常是大篇幅的，如果使用自动装配则省去了很多配置，并且如果对象的依赖发生更新我们可以不需要去更新配置，但是也带来了一定的缺点

自动装配的优点参考文档：

https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-autowire

缺点参考文档：

https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-autowired-exceptions



**自动装配的方法**

- no
- byName
- byType
- constructor

自动装配的方式参考文档：

https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-autowire

开启自动装配：

`default-autowire="byType`

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd"
        default-autowire="byType">

    <bean id="indexService" class="com.alonge.service.IndexService"></bean>
    <bean id="indexDao" class="com.alonge.dao.IndexDaoImpl"></bean>
</beans>
```

**byType 也是采用的setter的注入方式。**

spring 扫描xml中的类，创建到spring中，当发现某个对象有依赖时，从map中**根据类型**取依赖的对象。注入进去。

 `private IndexDao indexDao;`  根据类型

**byName也 是采用的setter的注入方式。**

byName根据的是set方法的名字：

​      ` public void setXxx(IndexDao indexDao) {` 

​      `<bean id="xxx" class="com.alonge.dao.IndexDaoImpl1"></bean>`

可以为每个bean单独指定`<bean id="xxx" class="com.alonge.dao.IndexDaoImpl" autowire="byType"></bean>`



**@Component和@Repository\@Service@\Controller 的区别**：

​	@Component是通用的，可以使用在任意类上，是@Repository\@Service@\Controller 的父类，但是官网上说@Repository\@Service@\Controller 在spring未来的版本中可能会有一些附加的用处。



**@Autowire 默认使用byType**，如果找不到就会按照byName方式自动装配。**按照属性名进行装配**

而**@Resource默认使用byName**，**按照属性名进行装配，而不是setXXX。**如果找不到就会按照byType方式自动装配。@Resource有一个type属性可以指定注入的类。



如果某个dao的实现有两个，都交给了spring管理，这时注入dao出错，可以用注解@Primary来指定。或者使用@Qualiffier

```java
@Repository
@Primary
public class IndexDaoImpl1 implements IndexDao {
```

```java
    @Autowired
    @Qualifier("indexDaoImpl1")
    private IndexDao indexDao;
```











**Spring懒加载**

官网：https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-lazy-init

不开懒加载的时候，spring容器启动的时候，就会初始化类，

开启懒加载后，在getBean的时候才会初始化类

```java
// 默认为true
@Lazy
public class IndexDaoImpl1 implements IndexDao {
```



**Spring作用域**

文档参考：

https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-scopes

![img](https://cdn.jsdelivr.net/gh/joelovealonge/noteimgs/image.png!original)

**使用@Scope注解指定作用域，默认是singleton**

​	**在singleton中注入prototype类，会导致prototype失去作用。**

​    因为容器值产生了一次singleton对象，只有一次机会设置属性dao。

​	那么有什么办法呢？

  1. 实现一个类，

  2. 使用@Lookup注解

     ```java
     @Service
     public abstract class IndexService1 {
     
     	// 值可以不取，取值主要是为了有多个IndexDao实现的时候确定是哪一个
         @Lookup("indexDao")
         public abstract IndexDao getIndexDao();
     
         public void service(){
             System.out.println(this.hashCode() + "=====this");
             System.out.println(getIndexDao().hashCode() + "======");
         }
     ```

     ```java
     @Repository("indexDao")
     @Scope("prototype")
     public class IndexDaoImpl implements IndexDao {
         @Override
         public void test() {
             System.out.println("test, I am a indexDaoImpl0");
         }
     }
     ```

     这洋就可以解决在singleton环境中，prototype作用域失效了。



**spring生命周期的回调**

参考文档：

https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-factory-lifecycle

三种方法实现：

- 实现`InitializingBean`接口，初始化实例之后的回调。
- 自定义一个方法    xml配置 init 
- @PostConstruct注解修饰一个自定义方法

销毁类似



**使类的扫描变快：**

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context-indexer</artifactId>
        <version>5.2.9.RELEASE</version>
        <optional>true</optional>
    </dependency>
</dependencies>
```

给类创建索引



Profile



**ctrl+alt+b  查看实现类**



**Bean的依赖：**

以集成mybatis为例：

mybatis专门为spring开发了一个支持spring bean的jar，简化了我们配置sqlSessionFactory。

引入依赖：

```xml
        <!-- mysql -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.5.0</version>
        </dependency>

        <!-- 提供了一个可以交给spring管理的mybatis Bean -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>2.0.0</version>
        </dependency>

        <!-- 测试数据源 spring-jdbc-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>5.1.5.RELEASE</version>
        </dependency>

        <!-- mysql -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.47</version>
        </dependency>
    </dependencies>
```

Appconfig

```java
@Configuration
@ComponentScan("com.alonge")
// @ImportResource("classpath:spring.xml")
public class AppConfig {

    @Bean
    public SqlSessionFactoryBean sqlSessionFactoryBean(DataSource dataSource) {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        return sqlSessionFactoryBean;
    }

    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource dataSource = new DriverManagerDataSource();
        dataSource.setDriverClassName("com.mysql.jdbc.Driver");
        dataSource.setUrl("jdbc:mysql://localhost:3306/test");
        dataSource.setUsername("root");
        dataSource.setPassword("root");
        return dataSource;
    }
}
```

上述SqlSeesionFactoryBean依赖DataSource，spring以前的版本需要在sqlSessionFactory方法上加上@Autowire,spring5之后不需要了，这个就是bean的依赖。

notes：

​	为什么mybatis提供了SqlSessionFactory，还要提供一个新的jar ----  mybatis-spring呢？  因为如果我们之间使用SqlSessionFactory需要在spring配置各种信息，还需要手动开启会话等等，提供一个springbean 后，我们只需要提供一个数据源即可。



