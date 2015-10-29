---
layout: post
title: "日志系统log4j切换logback总结"
description: ""
category: 技术分享
tags: [logback log4j]
---

#### log4j遇到的问题
线上log4j用的版本比较老，有时候遇到写日志的时候锁日志文件的问题，目前大多数web用于已经都在使用logback了他的好处有很多网上资料也比较多，他的性能提升主要是使用了java1.5之后的Concurrent包中的队列和列表，而log4j则使用的是ArrayList等实现，性能和并发性当然是大打折扣

#### 切换过程
切换过程就是参考webx文档[第 11 章 Webx日志系统的配置](http://openwebx.org/docs/logging.html)
首先要了解jcl、jul、slf4j、log4j、logback相互是什么关系，然后来进行排除冲突的jar包，下面这个图说明了他们之间的排斥关系
![](http://openwebx.org/docs/images/log/xlogback.png.pagespeed.ic.nr0Lfr9tG3.png)
因此就要在项目当中排除slf4j-log4j12这个jar包排除的过程非常简单，就是用mvn dependency:tree获取到当前间接依赖的jar包，当然如果这么简单就没必要记录了，下面来说说遇到的一些问题吧。

#### 遇到的问题
- 先说下如何排除吧在pom文件中，在项目的parent中添加下面的依赖,这会影响间接依赖
```
<dependencyManagement>
<!-- logger -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.5</version>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>jcl-over-slf4j</artifactId>
        <version>1.7.5</version>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>log4j-over-slf4j</artifactId>
        <version>1.7.5</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.1.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>commons-logging</groupId>
        <artifactId>commons-logging</artifactId>
        <version>99.0-does-not-exist</version>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>jul-to-slf4j</artifactId>
        <version>1.7.5</version>
    </dependency>
</dependencyManagement>
```
然后在子模块中需要引用jar包的地方添加下面的依赖，这会影响直接依赖
```
<!-- logger -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jul-to-slf4j</artifactId>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jcl-over-slf4j</artifactId>
</dependency>
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>log4j-over-slf4j</artifactId>
</dependency>
```
**然后在项目根目录下执行mvn dependency:tree 来排查哪些jar包间接依赖了slf4j-log4j12 这个包，这里需要注意的是不是运行一次就将所有的依赖，需要执行一次排查一个，因为maven判断当前项目如果有多个版本的同一个jar包，项目本身直接依赖的优先级会最高。此时maven自动会排除其它版本的，但是maven并不知道开发人员真正想要的。当你把这个优先级最高的排除了，此时maven又会选择优先级较低的那个，因此需要多次执行，这里推荐用eclipse比较直观**
如何排除webx文档上已经说明了，这里就不多说了。

- 我天真的以为这样就好了，屁颠屁颠的点开eclipse--->run as结果在webx启动的时候就提示
```
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/Users/Sunxc/.m2/repository/org/slf4j/slf4j-log4j12/1.7.5/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/Users/Sunxc/.m2/repository/ch/qos/logback/logback-classic/1.0.13/logback-classic-1.0.13.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
```
上面提示StaticLoggerBinder这个类存在多个jar包中，默认的他是加载第一个也就是slf4j-log4j12/1.7.5 这个版本，但是mvn dependency:tree的时候在项目中确实看不到了！！ 在eclipse中查看pom的Dependency Hierarchy查看发现子模块确实有这个jar包但是mvn dependency:tree的结果看不到，经过一番折腾我决定把该项目在本地删除（删除前提交svn）然后重新下下来 mvn eclipse:eclipse 后神奇的发现原来在eclipse中查看pom的Dependency Hierarchy中看到的那个jar包已经不存在了，当时有种立刻删除eclipse的感觉，再启动启动日志中已经提示加载/WEB-INF/logback.xml这个文件了

#### 再说下logback的配置文件
- 使用AsyncAppender异步输出日志
AsyncAppender本身并不会打印日志，而是要引用一个appender 并且只有有一个，他本身就是个异步队列。如果在类中通过LoggerFactory.getLogger(String name) 主要是String参数，获取logger ，想在日志中获取类名（不包括包路径）的pattern如下
```[%d{yyyy-MM-dd HH:mm:ss}] %-5level %C{0} # %m%n```
其中%C{0} 是获取类名忽略包路径，这里使用%C性能并不好并不推荐，这时获取的是LoggerFactory.getLogger(String name)  这个name的最后一个名字，如：在类CPSQuery.java 中通过LoggerFactory.getLogger("ops.static") 获取logger则在日志中打印的是是static 而不是CPSQuery这个类名，这里又经过一番折腾发现是使用AsyncAppender 就会这样，去掉后就可以获取类名了

### 至此log4j切换logback完成
如果业务中有依赖日志做业务分析、监控、恢复等需要多测试下最后附上log4j和logback的日志PatternLayout参数含义
[log4j](http://blog.csdn.net/guoquanyou/article/details/5689652)
[logback](http://aub.iteye.com/blog/1103685)
