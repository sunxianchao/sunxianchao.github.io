---
layout: post  
title: hadoop之mapreduce实现原理
category: 技术分享  
slug: hadoop-mapreduce-process
Tags: [mapreduce, hadoop]
---

学习hadoop是必须要掌握mapreduce的，目前看的书中似乎前面都是拿一些实例来讲解mapreduce的实现，但是里面的一些实现原理似乎并不是很明白。边看书，边思考，查了很多资料，目前有点小感觉就记录一下。  期间也查询了很多资料，后面会附上参考地址。
<!--break-->
###我是在示例上进行的分析，我直接复制下来。

1. 要输入的文件路径，在集群环境中是存储在hdfs上的，如果单个文件很大，就会进行切分默认（64M）并且存储在多个datanode节点上如果文件很大就会在2中创建多个InputSplit每个split对应一个map  
{%highlight java%}  
FileInputFormat.addInputPath(job, new Path(args[0]));
{%endhighlight%}

以下是发生在map阶段  
2. Mapper参数类型map的输入key、输入val（InputFormat类型），输出key、输出val（OutFormat类型）  
3. 会通过声明的输入key进行数据格式的定义，比如换行符分割，key为切割的数据长度，val是字符串数值，这个过程就是InputSplit  
4. 一个InputSplit就会去处理一个Block，也可以处理多个  
5. 循环读取文件，每次读取一行记录就会调用一次map函数，context.write会将数据写到内存缓冲区，如果缓冲区大小超过配置文件中设置的阀值，那么就会将数据同步到磁盘（后面的MemoryBuffe过程）以本示分析内存中存储的是{Tom, 1+Tom+Luck}....  
6. 接下来就是shuffle的过程了，这个过程又分为很多细节的过程  
  6.1. Partitioner（分区）这个过程就是决定由哪个reduce去处理（n 个reduce），对map输出的key对n取模得到m，那么就是第m个reduce处理生成这样的三个元素的数组{m, key, value}....  
  6.2. MemoryBuffer 内存缓冲区，每个map的结果和partition的结果都会放到内存缓存区默认是100M，阀值是0.8，如果超过80m那么就会同步到磁盘（后面的Spill）将6.1中生成的多个三元数组存放在缓冲区
  6.3. Spill 溢写过程，在6.2中如果数据达到阀值80M内存，Spill线程就会锁住这80M的数据缓冲区，将数据写到磁盘中，每次的Spill都会生成一个数据文件
每次Spill写到磁盘之前都会对数据key进行排序（Sort）以及合并（Combiner）的处理  
 6.4. Sort 排序过程，缓冲区的数据会按照key进行排序，如果是int类型的key那么就会数字降序排列，其他就是按照字母排序，那么相同的分区（m）就会存储在一块  
 6.5. Combiner 数据合并，如果设置了Combiner就会用combiner进行合并处理，通过此步的合并可以减少数据量的传输，Combiner函数实际上就是Reduce函数，但是Combiner是不能够处理sum、max 这类函数的（Combiner只应该用于那种Reduce的输入key/value与输出key/value类型完全一致，有的可能是要求value的形式也是一样的，比如本例中的value格式）  
 6.6. Merage on disk数据合并，map结束后会对Spill的文件进行多次合并，如有两个益写文件 f1：{Tom，1+Tom+Lucy} f2：{Tom，1+Tom+Jack}合并后的文件内容{Tom,1+Tom+Lucy|1+Tom+Jack},这个过程称为group，他是reduce的输入数据格式。（在这个过程中如果client设置过Combiner，也会使用Combiner来合并相同的key)  
 6.7. 至此map端的处理就结束了  

以下是reduce的过程
 7. 在map处理的过程中，reduce端就是不断的从map端拉取数据，至于reduce是如何发现map的，这个地方我想应该是通过JobTracker来感知的  
 8. copy数据到reduce端，从所有的map节点的端copy自己对应的partition的数据，数据来自多个map那么问题来了key有可能是相同的  
 9. 数据的合并merage，将多个益写文件进行合并处理，作为reduce的输入，将合并后的结果传给reduce函数来处理，这里的merage过程应该和map阶段的差不多，只是数据是来自不同的map而已，reduce端也会经过spill的处理，在这个阶段也会出现spill的过程，这个过程中如果你设置有Combiner，也是会启用内存达到阀值也会进行sort和combiner的过程，如果combiner对数据结果不会有影响那么就可以设置进行合并  
 10. 最终merage后的文件（在内存中或者是磁盘中）达到reduce函数，经过多次的spill的处理reduce输出的key是不会重复的。  
 11. reduce结束，得到最终的结果  
 12. 将结果输出到指定的hdfs路径下  
 
### 示例
示例是一个单表链接查询孙子和爷爷的过程数据格式：
{%highlight java%} 
child   parent
Tom     Lucy
Tom     Jack
Jone    Lucy
Jone    Jack
Lucy    Ben
Jack    Alice
Jack    Jesse
Terry   Alice
Terry   Jesse
Philip  Terry
Philip  Alma
Mark    Terry
Mark    Alma
Lucy    Mary
{%endhighlight%}

mapreduce分析过程
{%highlight java%} 
--------map阶段，每条结果对放在两个Map中-------
key                         value
Terry                       1+Philip+Terry
Alice                       1+Terry+Alice

key                         value
Philip                      2+Philip+Terry
Terry                       2+Terry+Alice

-------shuffle阶段,结果将作为reduce的输入-------
Alice                       1+Terry+Alice
Philip                      2+Philip+Terry
Terry                       1+Philip+Terry|2+Terry+Alice

--------reduce阶段，表1的child就是孙子，表2的parent就是爷爷--------
childArr                    parrentArr
Terry
                            Terry
Philip                      Alice
Philip                      Jesse

-----如果两个数组都不为空的话那么就进行一个笛卡尔积输出结果-------------
for(childArr){
    for(parentArr){
        childArr[i] parentArr[j]
    }
}
------结果-------
Philip  Alice
Philip  Jesse

{%endhighlight%}


marreduce实现代码
 
{%highlight java%} 
import java.io.IOException;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class SingleJoin {

    private static IntWritable times=new IntWritable(1);
    public static class SingleJoinMap extends Mapper<Object, Text, Text, Text> {

        @Override
        protected void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            String line=value.toString();
            int i=0;
            while(line.charAt(i) != ' ') {
                i++;
            }
            String[] values={line.substring(0, i).trim(), line.substring(i + 1).trim()};
            if(!values[0].equals("child")) {
                context.write(new Text(values[1]), new Text("1+" + values[0] + "+" + values[1]));
                context.write(new Text(values[0]), new Text("2+" + values[0] + "+" + values[1]));
            }

        }

    }
    public static class SingleJoinReduce extends Reducer<Text, Text, Text, Text> {

        @Override
        protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
            if(times.get() == 1) {
                context.write(new Text("grandchild"), new Text("grandparent"));
                times=new IntWritable(times.get() + 1);
            }
            List<String> children=new ArrayList<String>();
            List<String> parents=new ArrayList<String>();
            Iterator<Text> it=values.iterator();
            while(it.hasNext()) {
                String record=it.next().toString();
                String[] arr=record.split("\\+");
                int tbl=Integer.parseInt(arr[0]);
                String child=arr[1];
                String parent=arr[2];
                if(tbl == 1) {
                    children.add(child);
                } else {
                    parents.add(parent);
                }
            }
            if(children.size() > 0 && parents.size() > 0) {
                for(int i=0; i < children.size(); i++) {
                    for(int j=0; j < parents.size(); j++) {
                        context.write(new Text(children.get(i)), new Text(parents.get(j)));
                    }
                }
            }
        }

    }

    public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException {
        Configuration conf=new Configuration();
        Job job=Job.getInstance(conf, "sliglejoin");
        job.setJarByClass(SingleJoin.class);
        job.setMapperClass(SingleJoinMap.class);
        job.setCombinerClass(SingleJoinReduce.class);
        job.setReducerClass(SingleJoinReduce.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);

        FileInputFormat.addInputPath(job, new Path(args[0]));
        
        Path output=new Path(args[1]);
        FileSystem fs=FileSystem.get(conf);
        if(fs.exists(output)) {
            fs.delete(output, true);
        }

        FileOutputFormat.setOutputPath(job, output);
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}  
{%endhighlight%}
下面是这个示例


## 参考
1. [Hadoop mapreduce 过程简析 - SlideShare](http://www.slideshare.net/snakebbf/hadoop-mapreduce-12716482)
2. [MapReduce:详解Shuffle过程](http://langyu.iteye.com/blog/992916)