---
layout: post
title: java并发包之线程池ThreadPoolExecutor   
category: 技术分享
tagline: "Supporting tagline"
tags : [ThreadPoolExecutor,线程池]
---

##  为什么要用线程池  
1. 减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务  

2. 可以根据系统的承受能力，调整线程池中工作线线程的数目，防止因为消耗过多的内存，而把服务器累趴下(每个线程需要大约1MB内存，线程开的越多，消耗的内存也就越大，最后死机)

3. 在jdk1.5提供了线程池类ThreadPoolExecutor  

<!--break-->
##  ThreadPoolExecutor线程池的构造函数：  
{%highlight java%}  
new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, unit,runnableTaskQueue, handler);  
{%endhighlight%}  

参数解释：  

* corePoolSize：核心线程数量，当提交一个任务到线程池时,线程池会创建一个线程来执行任务,即使其他空闲的基本线程能够执行新任务也会创建线程,等到需要执行的任务数大于线程池基本大小时就不再创建  

* maximumPoolSize：线程池线程最大数量，线程池允许创建的最大线程数。如果队列满了,并且已创建的线程数小于最大线程数,则线程池会再创建新的线程执行任务  

* keepAliveTime：当一个线程无事可做，超过keepAliveTime 的时间时，线程池会判断，如果当前运行的线程数大于 corePoolSize，那么这个线程就被停掉。所以线程池的所有任务完成后，它最终会收缩到 corePoolSize 的大小。

* unit： Timeunit类型是一个枚举，表示 keepAliveTime 的单位  

* runnableTaskQueue： 用于保存等待执行的任务的阻塞队列。可以选择以下几个阻 塞队列  
   1. ArrayBlockingQueue:是一个基于数组结构的有界阻塞队列,此队列按 FIFO(先进先出) 原则对元素进行排序。  
   
   2. LinkedBlockingQueue:一个基于链表结构的阻塞队列,此队列按 FIFO (先进先出) 排 序元素,吞吐量通常要高于 ArrayBlockingQueue。静态工厂方法 Executors.newFixedThreadPool()使用了这个队列  
   
   3. SynchronousQueue:一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用 移除操作,否则插入操作一直处于阻塞状态,吞吐量通常要高于 LinkedBlockingQueue, 静态工厂方法 Executors.newCachedThreadPool 使用了这个队列  
   
  
## 线程池工作过程  

1. 当线程池对象刚刚创建时，线程池中是没有任何线程存在的，当有新的任务进来时才会创建  

2. 当执行execute添加线程任务时候，线程池先做如下判断：  

    * 
 
  