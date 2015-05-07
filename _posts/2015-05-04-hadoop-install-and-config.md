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
<!--break-->

## 1. docker安装hadoop镜像
hadoop镜像我是在[hadoop-docker](https://github.com/sequenceiq/hadoop-docker)由于网络环境不是很好，我事先把hadoop-docker-2.6.0.tar.gz 还有jdk1.7都下载好了。  
git clone https://github.com/sequenceiq/hadoop-docker.git  
下载下来的项目 把下载好的文件都丢到这个项目里 修改了Dockerfile把这两个文件wget 的地方都修改成 ADD 我修改后的文件如下：
{%highlight c%}
ADD jdk-7u51-linux-x64.rpm .
ADD hadoop-2.6.0.tar.gz /usr/local/
{%endhighlight%}  
       
随后就可以进行docker build了，这个项目的README上已经给了提示按照上面的操作就可以，漫长的等待之后就总是惊喜不断，期间无数次断网，下载超时等问题，最后经过2个多小时终于完成了。

## 2. 启动容器制作自己的镜像
{%highlight c%}
docker run -i -t sequenceiq/hadoop-docker:2.6.0 /etc/bootstrap.sh -bash
{%endhighlight%} 
上面的命令将直接运行/etc/bootstrap.sh并启动hadoop。
修改/usr/local/hadoop/etc/hadoop/core-site.xml
{%highlight c%}
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://master:9000</value>
    </property>    
</configuration>
{%endhighlight%} 
我们在启动其他hadoop slave的时候这个地方是需要都是写成master的地址，所以我们在这边统一修改
修改/etc/bootstrap.sh
将sed那一行注视点否则每次运行这个脚本都会被替换的
退出容器后  
{%highlight c%}
docker commit -a 'sunxc@vip.qq.com' -m 'add hadoop/bin to PATH' cfeb92e4eb3c  d
evops/hadoop2.6:v0.3
{%endhighlight%}   
制作我们自己的镜像文件 
{%highlight c%} 
docker@boot2docker:~$ docker commit -a 'sunxc@vip.qq.com' -m 'add hadoop/bin to PATH' cfeb92e4eb3c  d
evops/hadoop2.6:v0.3
ef9da5d858554d0d20546088cb0d20e5239ae5f4c8b36d5a1bd94c0d980698d7
docker@boot2docker:~$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
devops/hadoop2.6    v0.3                ef9da5d85855        5 seconds ago       1.708 GB
devops/hadoop2.6    v0.2                cf03aad60ef3        4 minutes ago       1.708 GB
devops/hadoop2.6    v0.1                ecaca1ea44f5        2 days ago          1.708 GB
sequenceiq/pam      centos-6.5          2e0b6343afc8        10 weeks ago        923.5 MB
{%endhighlight%} 

## 3. 启动集群环境的容器
hadoop 集群的基本要求,其中一个是 master 结点,主要是用于运行 hadoop 程序中的 namenode、secondorynamenode 和 jobtracker（新版本名字变了） 任务。用外两个结点均为 slave 结点,其中一个是用于冗余目的,如果没有冗 余,就不能称之为 hadoop 了,所以模拟 hadoop 集群至少要有 3 个结点。这里我们先启动容器 不执行/etc/bootstrap.sh 这个脚本。  

这里有一个问题：
Docker容器中的ip地址是启动之后自动分配的，且不能手动更改
hostname、hosts配置在容器内修改了，只能在本次容器生命周期内有效。如果容器退出了，重新启动，这两个配置将被还原。且这两个配置无法通过commit命令写入镜像
我们搭建集群环境的时候，需要指定节点的hostname，以及配置hosts。
这里只为学习，就手动修改hosts。只不过每次都得改，这里不知道如何处理，docker也是刚刚入门如果有知道的请高手指点一下！！

启动容器
{%highlight c%} 
docker run -p 9000:9000 -p 50070:50070 -p 8088:8088 -p 50010:50010 -p  50020:50020 -p 50030:50030 -h master --name master_host -it devops/hadoop2.6:v0.5 /bin/bash

docker run -h slave --name slave_host -it devops/hadoop2.6:v0.5 /bin/bash

docker run -h slave2 --name slave2_host -it devops/hadoop2.6:v0.5 /bin/bash
{%endhighlight%} 

进入每个容器ifconfig 查看ip地址信息，进行hosts文件配置
vi /etc/hosts
172.17.0.12    master
172.17.0.13    slave
172.17.0.14    slave2

在master节点上进入HADOOP_HOME/etc/hadoop/ 
vi slaves
添加两个slave主机名 保存退出
然后启动master上的节点执行/etc/bootstrap.sh
可以看见相关的日志信息
在master节点上通过命令hdfs dfsadmin -report查看DataNode是否正常启动

## 4.出现过的问题
1. 在master上执行/etc/bootstrap.sh后 查看两个slave 发现datanode没有启动，查看日志如下信息：  
{%highlight c%}
2015-05-06 22:53:45,766 INFO org.apache.hadoop.hdfs.server.common.Storage: Lock on /tmp/hadoop-root/dfs/data/in_use.lock acquired by nodename 3886@slave
2015-05-06 22:53:45,769 FATAL org.apache.hadoop.hdfs.server.datanode.DataNode: Initialization failed for Block pool <registering> (Datanode Uuid unassigned) service to master/172.17.0.2:9000. Exiting.
java.io.IOException: Incompatible clusterIDs in /tmp/hadoop-root/dfs/data: namenode clusterID = CID-4d002d3c-f5c9-4004-8e23-3705553c3dbc; datanode clusterID = CID-f3bcc414-d2dd-4fed-ba67-34bb71f24e2f
	at org.apache.hadoop.hdfs.server.datanode.DataStorage.doTransition(DataStorage.java:646)
	at org.apache.hadoop.hdfs.server.datanode.DataStorage.addStorageLocations(DataStorage.java:320)
	at org.apache.hadoop.hdfs.server.datanode.DataStorage.recoverTransitionRead(DataStorage.java:403)
	at org.apache.hadoop.hdfs.server.datanode.DataStorage.recoverTransitionRead(DataStorage.java:422)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.initStorage(DataNode.java:1311)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.initBlockPool(DataNode.java:1276)
	at org.apache.hadoop.hdfs.server.datanode.BPOfferService.verifyAndSetNamespaceInfo(BPOfferService.java:314)
{%endhighlight%}  

进入/tmp/hadoop-root/dfs/ 将下面三个目录data  name  namesecondary当中的子目录删除，不要删除这三个目录本身
然后重新hdfs namenode -format 即可

2. 如果是执行master 上的/etc/bootstrat.sh 那么两个slave上的sshd 是需要手动执行的 否则是master 是连不上slave的 在两个slave上执行 
{%hightlight c%}
service sshd start
{%endhighlight%}
