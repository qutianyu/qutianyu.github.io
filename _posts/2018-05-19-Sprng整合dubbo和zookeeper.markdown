---
layout:     post
title:      "Sprng整合dubbo和zookeeper"
subtitle:   "Java工程学习"
date:       2018-05-19
author:     "QuTianyu"
header-img: "img/post-build-a-blog.jpg"
tags:
    - java
    - spring
---


**版权声明：欢迎分享，转载请注明出处。**

---


这几天研究了一下分布式框架，具体创建项目和项目的结构目录就不提了，资料很多，千篇一律，重点记录一下自己在spring整合dubbo和zookeeper中趟过的坑。
版本上，ide使用idea2018.1，spring使用的是5.0.7，dubbo使用2.6.1，zookeeper使用的是3.5.3。这些都是截止到2018年5月18日的最新版本，经过测试，spring换成4.x，dubbo换成2.5.x，zookeeper换成3.4.x也都没有问题。

## 母项目
首先在pom里添加spring全家桶和dubbo、zookeeper的依赖（真实开发里可能会用到更多jar包的依赖，这里只是demo）。
```xml
<properties>
	<spring.version>5.0.7.BUILD-SNAPSHOT</spring.version>
	<dubbo.version>2.6.1</dubbo.version>
	<zookeeper.version>3.4.7</zookeeper.version>
</properties>

<dependencies>
    <!--spring-->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>${spring.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context-support</artifactId>
        <version>${spring.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-core</artifactId>
        <version>${spring.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-beans</artifactId>
        <version>${spring.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>${spring.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jdbc</artifactId>
        <version>${spring.version}</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-tx</artifactId>
        <version>${spring.version}</version>
    </dependency>
    <!--springmvc包-->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>${spring.version}</version>
    </dependency>
	<!--dubbo-->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>dubbo</artifactId>
        <version>${dubbo.version}</version>
        <exclusions>
            <exclusion>
                <artifactId>spring</artifactId>
                <groupId>org.springframework</groupId>
            </exclusion>
            <exclusion>
                <artifactId>netty</artifactId>
                <groupId>org.jboss.netty</groupId>
            </exclusion>
        </exclusions>
    </dependency>
    <!--zookeeper-->
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>${zookeeper.version}</version>
    </dependency>
</dependcies>
```
**在引入dubbo的时候务必要排除掉dubbo里spring的依赖，因为dubbo 2.x本身依赖的是一个2.x 版本的spring，会和项目里的高版本spring冲突。另外，zookeeper的jar包不能引3.53-Beta，会出兼容性问题**

##消费方（Controller）
在web.xml里添加spring配置文件applicationContext.xml，这个配置文件在src/main/resource文件夹下。
```xml
<servlet>
    <servlet-name>webapp</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:applicationContext.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>webapp</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```
**其实配置方式很多，这里用的比较传统的方法。**

然后在applicationContext.xml里扫描bean，并配置dubbo和zookeeper。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns:context="http://www.springframework.org/schema/context"
   xmlns:task="http://www.springframework.org/schema/task"
   xmlns:mvc="http://www.springframework.org/schema/mvc"
   xmlns:tx="http://www.springframework.org/schema/tx"
   xmlns:aop="http://www.springframework.org/schema/aop"
   xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
   xsi:schemaLocation="http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans.xsd
   http://www.springframework.org/schema/context
   http://www.springframework.org/schema/context/spring-context.xsd
   http://www.springframework.org/schema/mvc
   http://www.springframework.org/schema/mvc/spring-mvc.xsd
   http://www.springframework.org/schema/tx
   http://www.springframework.org/schema/tx/spring-tx.xsd
   http://www.springframework.org/schema/aop
   http://www.springframework.org/schema/aop/spring-aop.xsd
   http://www.springframework.org/schema/task
   http://www.springframework.org/schema/task/spring-task-3.2.xsd
   http://code.alibabatech.com/schema/dubbo
   http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

   <!--扫描org.demo.controller路径下所有带有@Controller标签的类-->
   <!--目前该路径下有一个DemoController.java-->
   <context:component-scan base-package="org.demo.controller"/>

   <!--配置dubbo的名字，随便取，不重复即可-->
   <dubbo:application name="dubbo_controller"/>
   <!-- 配置zookeeper，address里输入zookeeper所在计算机的ip和端口（此处为本机） -->
   <dubbo:registry  protocol="zookeeper" address="127.0.0.1:2181" />
   <!-- 把供给方提供的DemoService.java提取出来 -->
   <dubbo:reference id="demoService" interface="org.demo.service.DemoService" check="false"/>
</beans>
```
**不能忘记在xml的头部配置dubbo的值域。**

##攻击方（Service）
web.xml配置雷同，仍然是配置applicationContext.xml。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns:context="http://www.springframework.org/schema/context"
   xmlns:task="http://www.springframework.org/schema/task"
   xmlns:mvc="http://www.springframework.org/schema/mvc"
   xmlns:tx="http://www.springframework.org/schema/tx"
   xmlns:aop="http://www.springframework.org/schema/aop"
   xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
   xsi:schemaLocation="http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans.xsd
   http://www.springframework.org/schema/context
   http://www.springframework.org/schema/context/spring-context.xsd
   http://www.springframework.org/schema/mvc
   http://www.springframework.org/schema/mvc/spring-mvc.xsd
   http://www.springframework.org/schema/tx
   http://www.springframework.org/schema/tx/spring-tx.xsd
   http://www.springframework.org/schema/aop
   http://www.springframework.org/schema/aop/spring-aop.xsd
   http://www.springframework.org/schema/task
   http://www.springframework.org/schema/task/spring-task-3.2.xsd
   http://code.alibabatech.com/schema/dubbo
   http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

   <!-- 扫描org.demo.service路径下所有带有@Service标签的类 -->
   <!-- 目前该路径下有一个DemoService.java -->
   <context:component-scan base-package="org.demo.service"/>


   <!-- 配置dubbo的名字，随便取，不重复即可 -->
   <dubbo:application name="dubbo_service"/>
   <!-- 配置zookeeper，address里输入zookeeper所在计算机的ip和端口（此处为本机） -->
   <dubbo:registry  protocol="zookeeper" address="127.0.0.1:2181" />
   <!-- 用dubbo协议在20880端口暴露服务 -->
   <dubbo:protocol name="dubbo" port="20880" />
   <!-- 声明需要暴露的接口 -->
   <dubbo:service interface="org.demo.service.DemoService" protocol="dubbo" ref="demoService" />
</beans>
```
**供给方service的@Service注解需要带一个和ref值相同的value值，比如DemoService的注解需要写作@Service("demoService")。不写value的话会扫描不到。**