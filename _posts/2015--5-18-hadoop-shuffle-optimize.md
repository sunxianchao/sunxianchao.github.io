### shuffle优化
1. mapreduce的过程并不是最优的，因为他每次都需要对key进行排序操作，如果我们要的结果并不在意顺序那么就是浪费，这里如何优化？？  
2. io操作一定是耗时比较久的，缓冲区内容超过阀值就会写磁盘，可以设置io.sort.mb的值，来减少写入磁盘的次数。  
3. 在reduce端内存很小的时候可以设置mapred.inmem.merge.threshold 设置位0，将mapred.job.reduce.input.buffer.percent设置为1，来减少io的次数。  
4. 如果由于机器负载过高，导致job的执行速度慢于其他job的执行平均速度，那么jobracker就会启动在集群当中启动一个新的job，如果原job和新job有执行完成的，那么jobracker就会把另外一个kill掉，这个策略叫推行测试，hadoop默认是启用的，但是对于代码上的漏洞是无法感知的。因此会对集群造成负担  
5. jvm重用，map和reduce都是在tasktracker虚拟机上运行的（map、reduce是在TaskTracker的子进程中运行的）分配一个job就会启动一个java虚拟机，如果有大量零碎的文件输入那么启动的虚拟机如果考虑重用的话，此处是可以进行一定优化的，mapred.job.reuse.jvm.num.tasks 默认是1，每个任务就会启动一个jvm，设置-1是不受数据的限制。   
6. 跳过坏记录，由于hadoop处理的数据非常庞大，如果某个处理中出现错误，hadoop就去尝试重新执行（默认4次）但是失败后会导致整个job失败，所以此处可以设置 跳过坏记录，从而保证整个job的运行  
7. Slot是task执行的逻辑概念，可以理解为TaskTracker同时并发可执行多少个task的能力。Slot分为map slot和reduce slot，可由用户通过mapreduce.tasktracker.map.tasks.maximum和mapreduce.tasktracker.reduce.tasks.maximum分别设置，默认分别为2，这个值的设置是要根据系统cup核数还有datanode等来计算的。

参考
1. [MapReduce: JT默认task scheduling策略](http://langyu.iteye.com/blog/910677)