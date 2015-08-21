---
layout: post
title: "maven 配置 setting.xml 中资源库的问题"
description: ""
category:  maven
tags: [maven]
---

###maven项目共用工程依赖问题

joysdk_common 这样的工程需要很多项目引用，因此需要将编译后的jar包上传到maven仓库中，以便团队相互协作。

nexus后台，选择对应的仓库后在artifact upload中上传;
在项目顶层的pom.xml中设置distributionManagement节点，节点内容如下：
<!--break-->
{% highlight xml %} 
<distributionManagement>     
  <repository> 
    <id>Releases</id> 
    <name>Releases</name> 
    <url>http://192.168.11.44:8081/nexus/content/repositories/releases/</url> 
  </repository> 
  <snapshotRepository> 
    <id>Snapshots</id> 
    <name>Snapshots</name> 
    <url>http://192.168.11.44:8081/nexus/content/repositories/snapshots/</url>
  </snapshotRepository> 
</distributionManagement>
{% endhighlight %}

在maven 安装目录中的conf/setting.xml中设置对应的仓库用户名和密码 
{% highlight xml %} 
<servers>
<server>
    <id>Releases</id>
    <username>deployment</username>
    <password>deployment123</password>
</server>
<server>
   <id>Snapshots</id>
   <username>deployment</username>
   <password>deployment123</password>
</server>
</servers>
{% endhighlight %}
其中pom.xml中的repository节点id对应的名字要和setting中server节点中的id一致否则会提示授权失败的异常信息 
{% highlight xml %} 
Caused by: org.apache.maven.artifact.deployer.ArtifactDeploymentException: Failed to deploy   
artifacts: Could not transfer artifact com.joysdk:joysdk_common:jar:0.0.1-20150619.074651-1 from/to 
LocalServer http://192.168.11.44:8081/nexus/content/repositories/snapshots/: Failed to transfer 
file: http://192.168.11.44:8081/nexus/content/repositories/snapshots/com/joysdk/joysdk_common/0.0.1-
SNAPSHOT/joysdk_common-0.0.1-20150619.074651-1.jar. Return code is: 401, ReasonPhrase: Unauthorized.
{% endhighlight %}
发布jar包

以上信息可以放在顶层的pom中，在子工程的pom中的build节点添加deploy 默认goal
如添加默认goal 则执行mvn 即可，否则需要指定 mvn goal