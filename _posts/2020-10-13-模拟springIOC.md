---
title: 模拟springIOC
tags:

  - Spring源码
---

模拟springIOC

```
1. 哪些类需要我来关联
2. 怎么告诉我这些类（写bean）
3. 怎么维护依赖关系（setter、construct）
4. 怎么实现setter或者construct
```

xml实现以及注解实现

详见我的github：



### Spring beanFactory和factoryBean的区别

**BeanFactory是IOC最基本的容器，负责产生bean**，Beanfactory提供了容器最基本的形式，给具体的IOC容器的实现提供了规范。有个重要的方法getBean。

**FactoryBean是一个接口，当在IOC容器中的Bean实现了FactoryBean后，通过getBean(String beanName) 获取到的对象并不是FactoryBean的实现类对象**，而是这个实现类中的getObject方法返回的对象。要获取FactoryBean的实现类，就要getBean(&beanName)，在beanName之前加上&。



FactoryBean 有什么作用呢？ 

简单来说FactoryBean为IOC容器中Bean的实现提供了更加灵活的方式，我们可以在getObject()方法中灵活配置。

比如mybatis提供的SqlSessionFactory类，如果我们用该类进行配置，各种属性各种依赖，手动开启事务等，这种复杂的类，配置很难维护。

为此mybatis专门为spring提供了一SqlSessionFactoryBean