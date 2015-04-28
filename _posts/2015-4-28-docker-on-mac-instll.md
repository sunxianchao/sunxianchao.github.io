---
layout: post  
title: 在mac上安装使用docker  
category: 技术分享  
slug: mac-install-docker  
Tags: [docker]
---

## Docker简介

Docker就是一个应用程序执行容器，类似虚拟机的概念。但是与虚拟化技术的不同点在于下面几点：

1. 虚拟化技术依赖物理CPU和内存，是硬件级别的；而docker构建在操作系统上，利用操作系统的containerization技术，所以docker甚至可以在虚拟机上运行。  
2. 虚拟化系统一般都是指操作系统镜像，比较复杂，称为“系统”；而docker开源而且轻量，称为“容器”，单个容器适合部署少量应用，比如部署一个redis、一个memcached。  
3. 传统的虚拟化技术使用快照来保存状态；而docker在保存状态上不仅更为轻便和低成本，而且引入了类似源代码管理机制，将容器的快照历史版本一一记录，切换成本很低。  
4.传统的虚拟化技术在构建系统的时候较为复杂，需要大量的人力；而docker可以通过Dockfile来构建整个容器，重启和构建速度很快。更重要的是Dockfile可以手动编写，这样应用程序开发人员可以通过发布Dockfile来指导系统环境和依赖，这样对于持续交付十分有利。  
5. Dockerfile可以基于已经构建好的容器镜像，创建新容器。Dockerfile可以通过社区分享和下载，有利于该技术的推广。

<!-break->

##Docker的主要特性
（摘自Docker：具备一致性的自动化软件部署)：

1. 文件系统隔离：每个进程容器运行在完全独立的根文件系统里。
2. 资源隔离：可以使用cgroup为每个进程容器分配不同的系统资源，例如CPU和内存。
3. 网络隔离：每个进程容器运行在自己的网络命名空间里，拥有自己的虚拟接口和IP地址。
4. 写时复制：采用写时复制方式创建根文件系统，这让部署变得极其快捷，并且节省内存和硬盘空间。
5. 日志记录：Docker将会收集和记录每个进程容器的标准流（stdout/stderr/stdin），用于实时检索或批量检索。
6. 变更管理：容器文件系统的变更可以提交到新的映像中，并可重复使用以创建更多的容器。无需使用模板或手动配置。
7. 交互式Shell：Docker可以分配一个虚拟终端并关联到任何容器的标准输入上，例如运行一个一次性交互shell。

目前Docker正处在开发阶段，官方不建议用于生产环境。另外，Docker是基于Ubuntu开发的，所以官方推荐将其安装在Ubuntu的操作系统上，目前只能安装在linux系统上。

## Docker安装与配置
由于Docker引擎是使用了特定于Linux内核的特性，所以需要安装一个轻量级的虚拟机（如VirtualBox）来在OSX上运行。所以需要下载官方提供的[Boot2Docker](https://github.com/boot2docker/boot2docker)来运行Docker守护进程  

1. 找到最新的Release版本，https://github.com/boot2docker/osx-installer/releases， 下载PKG，直接安装.

2. 安装完毕后就可以使用boot2docker命令来操作vm中的相关操作
3. 初始化：boot2docker init，运行后可以看到如下日志信息  
{% highlight shell%}  
Latest release for boot2docker/boot2docker is v1.4  
Downloading boot2docker ISO image...  
Success: downloaded https://github.com/boot2docker/boot2docker/releases/download/v1.4/boot2docker.iso  
to /Users/shengli/.boot2docker/boot2docker.iso  
Generating public/private rsa key pair.  
Your identification has been saved in /Users/shengli/.ssh/id_boot2docker.  
Your public key has been saved in /Users/shengli/.ssh/id_boot2docker.pub.  
The key fingerprint is:  
ff:7a:53:95:e6:44:27:70:e1:ac:0a:b5:02:35:72:29 Sunxc@192.168.1.104
The key's randomart image is:  
+--[ RSA 2048]----+  
{% endhighlight %}  
从上面的日志可以看到的他会下载docker的镜像文件，然后生成密钥用于ssh登录使用

4. 启动 boot2docker start  
{% highlight shell %}  
Sunxc:~$ boot2docker start
Waiting for VM and Docker daemon to start...
.............ooo
Started.
Writing /Users/Sunxc/.boot2docker/certs/boot2docker-vm/ca.pem
Writing /Users/Sunxc/.boot2docker/certs/boot2docker-vm/cert.pem
Writing /Users/Sunxc/.boot2docker/certs/boot2docker-vm/key.pem

To connect the Docker client to the Docker daemon, please set:
    export DOCKER_TLS_VERIFY=1
    export DOCKER_HOST=tcp://192.168.59.103:2376
    export DOCKER_CERT_PATH=/Users/Sunxc/.boot2docker/certs/boot2docker-vm  
{% endhighlight %} 
按照他的提示添加以上3个变量  
5. 查看版本信息 boot2docker version  
{% highlight shell %}  
Sunxc:~$ docker version  
Client version: 1.4.0  
Client API version: 1.16  
Go version (client): go1.3.3  
Git commit (client): 4595d4f  
OS/Arch (client): darwin/amd64  
Server version: 1.4.0  
Server API version: 1.16  
Go version (server): go1.3.3  
Git commit (server): 4595d4  
{% endhighlight %}  
7. 下载第一个docker应用镜像 docker版的helloworld  
 {% highlight shell %} 
docker@boot2docker:~$ docker run hello-world
Unable to find image 'hello-world:latest' locally
Pulling repository hello-world
91c95931e552: Download complete
a8219747be10: Download complete
Status: Downloaded newer image for hello-world:latest
Hello from Docker.
{% endhighlight %}
输出Hello from Docker.说明我们的docker安装是没有问题的

##Docker 实战 nginx 安装
docker run --rm -i -t -p 80:80 nginx  
经过很多的download就下载并启动了一个nginx服务了
以上命令是直接运行，如果没有会自动拉取一个镜像其实是这样执行的
docker pull registry.hub.docker.com/ubuntu:12.04
从指定的仓库下载ubuntu标记位12.04的版本，后面的标记如果不指定则下载最新的

## 常用的docker命令
1. 查看安装了哪些镜像 docker iamges
2. 删除镜像 docker rmi imageID 
3. 删除人容器 docker rm containID
4. 查看docker上运行的进程 docker ps -a
5. 停止某个容器 docker stop containID

