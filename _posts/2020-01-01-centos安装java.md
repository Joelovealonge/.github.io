---
title: centos安装java
tags:

  - Java基础
---



##### 一、下载java

http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html

找到下载地址然后用`wget`命令下载tar.gz包

##### 二、安装

1. 创建安装目录

   ```
   mkdir /usr/local/java
   ```

2. 解压至安装目录

   ```
   tar -zxvf jdk-8u271-linux-x64.tar.gz -C /usr/local/java/
   ```

   

##### 三、设置环境变量

1. 打开文件`etc/profile`

2. 在末尾添加

   ```
   export JAVA_HOME=/usr/local/java/jdk1.8.0_271
   export JRE_HOME=${JAVA_HOME}/jre
   export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
   export PATH=${JAVA_HOME}/bin:$PATH
   ```

3. 重启配置文件

   ```
   source /etc/profile
   ```

4. 检查

   ```
   java -version
   ```

   

##### Centos8不能显示中文

```
yum install glibc-common

yum install -y langpacks-zh_CN

vim /etc/locale.conf # 修改这个文件 LANG=zh_CN.utf8
```

