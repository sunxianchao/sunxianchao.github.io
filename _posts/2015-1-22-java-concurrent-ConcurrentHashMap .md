---
layout: post
title: java并发包之ConcurrentHashMap   
category: 技术分享
tagline: "Supporting tagline"
tags : [ConcurrentHashMap,并发]
---



### 1. 先说下HashTable和HashMap

线程不安全的 HashMap，是不能用于并发的情况因为他不是线程安全的，在多线程的环境下调用put 操作会引起死循环,导致 CPU 利用率接近 100%,所以在并发情况下不能使用 HashMap  

线程安全的 HashTable，虽然是现成安全的但是效率低下，他内部是使用synchronized 来保证线程安全， 当一个线程访问 HashTable 的同步方法时,其他线程也要去访问同步方法时,就会进入阻塞状态。
**如线程 1 使用 put 进行添加元素,线程 2 不但不能使用 put 方法添加元素,并且也不能使用 get 方法来获取元素,所以竞争越激烈效率越低**  

<!--break-->

### 2. ConcurrentHashMap的锁的原理  

之所以ConcurrentHashMap的性能高是因为他内部并没有使用synchronized进行同步，而是将里面的数据进行分段加锁处理，首先他将数据分成一段一段的存储, 然后给每一段数据配一把锁, 当一个线程占用锁去访问其中一个段数据的时候, 其他段的数据也能被其他线程访问。原理很简单但是里面的实现还是很复杂的。

1. ConcurrentHashMap 结构：  
    
    ![ConcurrentHashMap 结构](/images/concurrenthashmap-structure.png)  
    
    ConcurrentHashMap中的数据是由Segment数组和HashEntry数组组成的，其中Segment 是ReentrantLock（可重用锁的）的子类，而HashEntry 是实际存储数据的类，实际上是一个链表结构。一个ConcurrentHashMap 包含一个Segment数组，一个Segment中包含一个HashEntry数组，每个Segment锁守护一个HashEntry数组，当对某一个数据进行修改的时候，首先要获取这租数据的Segment才可以。

2. 初始化方法：  

{% highlight java%}
	// 默认初始容量
    static final int DEFAULT_INITIAL_CAPACITY = 16;
    
 	// 默认增长因子，当 table 中包含元素的个数超过了 table 数组的长度与装载因子的乘积时，将进行一次扩容。
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
	// 默认的并发等级，即是并发数量。
    static final int DEFAULT_CONCURRENCY_LEVEL = 16;
	
	// 允许的最大容量，由于要求该值必须是2的指数，并且是int类型，所以1<<30是其能用的最大值。
    static final int MAXIMUM_CAPACITY = 1 << 30;
	
	// 默认每一个segment 的容量，必须是2的指数，至少为2， 否则分段锁失效。
    static final int MIN_SEGMENT_TABLE_CAPACITY = 2;
	
	// 最大的 segment  数量，必须小于2的24次方，
    static final int MAX_SEGMENTS = 1 << 16; 
	
	// 在加锁前重试的次数
    static final int RETRIES_BEFORE_LOCK = 2;

{% endhighlight %}  

默认的配置初始化时的容量为16，当达到12个的时候就会扩容，并发数量是16个。  
3. get操作：  
    
   get 操作的高效之处在于整个 get 过程不需要加锁，除非是读取到了空值才会加重读锁，因此尽量避免向ConcurrentHashMap中存储空值的情况。get不加锁是因为在HashEntry中的value变量被定义成了volatile类型，因此可以保证了变量在线程之间的可见性，能够被多线程同时读,并且保证不会读到过期的值，对 volatile 字段的写入操作先于读操作,即使两个线程同时修改和获取 volatile 变量,get 操作 也能拿到最新的值。  
4. put 操作：  
   
   在多线程环境下对变量进行写操作是必须要加锁的，因此在put 的时候会先定位并获取Segment锁，然后进行插入操作，插入操作首先要检查Segment对应的HashEntry数组是否需要扩容，然后定位到存储数据的HashEntry进行操作。  
5. size 操作：

   要获取ConcurrentHashMap中元素大小，统计Segment中的有一个count和一个modCount变量，但是单纯的统计count然后相加是不可以的，如果有一个线程修改count，那么就将获取不准确的count值，Segment是通过另外一个变量modCount来统计的在 put , remove 和 clean 方法里操作元素前都会将变量 modCount 进行加 1  这样就知道容器中的内容是否有变化。通过两次获取count就可以知道这个值是否准确。 

