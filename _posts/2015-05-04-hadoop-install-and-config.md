---
layout: post  
title: 在docker上搭建hadoop分布式集群环境
category: 技术分享  
slug: docker-setup-hadoop
Tags: [docker, hadoop]
---

hadoop刚开始学习，想在docker上进行hadoop的分布式集群环境的搭建，于是小试牛刀，可能因为对docker不是很熟悉，还是遇见了一些小问题，但是现在想想可能还是不够细心的缘故吧，特此记录一下。

下面是学习hadoop整理的一点笔记 就放在这里好了  
### hadoop 分别从3个角度将主机划分为两种角色：  
1. 划分位master、slaver  
2. 从HDFS角度划分位NameNode（目录的管理者）和DataNode  
3. 从mapreduce角度分位JobTracker和TaskTracker  

### 伪分布式搭建  
1. 在hadoop-env.sh中指定 JAVA_HOME  
2. conf/core-site.xml hadoop核心配置文件中修改hdfs的ip和端口  
3. conf/hdfs-site.xml 这是hdfs的配置文件单机版是1 备份版是是3 默认  
4. conf/mapred-site.xml 这是mapreduce的配置文件，配置JobTracker的端口和地址  

### 格式化并启动
1. bin/hadoop NameNode -format 格式化文件系统
2. bin/start-all.sh启动
<!-break->

## 1. docker安装hadoop镜像
hadoop镜像我是在[hadoop-docker](https://github.com/sequenceiq/hadoop-docker)由于网络环境不是很好，我事先把hadoop-docker-2.6.0.tar.gz 还有jdk1.7都下载好了。  
git clone https://github.com/sequenceiq/hadoop-docker.git  
下载下来的项目 把下载好的文件都丢到这个项目里 修改了Dockerfile把这两个文件wget 的地方都修改成 ADD 我修改后的文件如下：
{%highlight shell%}
ADD jdk-7u51-linux-x64.rpm .
ADD hadoop-2.6.0.tar.gz /usr/local/
{%endhighlight%}  
       
随后就可以进行docker build了，这个项目的README上已经给了提示按照上面的操作就可以，漫长的等待之后就总是惊喜不断，期间无数次断网，下载超时等问题，最后经过2个多小时终于完成了。

