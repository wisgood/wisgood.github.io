---
layout: post
title:  "mapreduce中split划分"
categories: Spark
tags:  Hadoop MapReduce 
author: wisgood
---


* content
{:toc} 

面试的过程中，笔者经常喜欢问一个问题：hadoop中map数是怎么确定的？但发现还是有好多面试者都答不上来。这个问题其实算是比较基础的一个问题，对于理解mapreduce的原理很有帮助。

今天有空结合源码分析一下。
本文以hadoop2.7.2的版本作为分析，代码链接如下。 —— <a href="https://github.com/apache/hadoop/tree/release-2.7.3-RC2/hadoop-mapreduce-project/hadoop-mapreduce-client/hadoop-mapreduce-client-core/src/main/java/org/apache/hadoop" target="_blank"> [ github代码地址 ]

本文以org.apache.hadoop.mapreduce包作为讲解。org.apache.hadoop.mapred包划分的过程稍有不同。

# 1、map个数的确定
map的个数等于split的个数。我们知道，mapreduce在处理大文件的时候，会根据一定的规则，把大文件划分成多个，这样能够提高map的并行度。
划分出来的就是InputSplit，每个map处理一个InputSplit.因此，有多少个InputSplit，就有多少个map数。


# 2、谁负责划分split
主要是InputFormat。InputFormat类有2个重要的作用：

- 1）将输入的数据切分为多个逻辑上的InputSplit，其中每一个InputSplit作为一个map的输入。
- 2）提供一个RecordReader，用于将InputSplit的内容转换为可以作为map输入的k,v键值对。

InputFormat在新版的实现是一个抽象类，其继承关系如下:
![这里写图片描述](https://images2015.cnblogs.com/blog/760432/201510/760432-20151027105804044-1980762420.png)

从上图我们看到FileInputFormat是使用比较广泛的类，输入格式如果是hdfs上的文件，基本上用的都是FileInputFormat的子类，如TextInputFormat用来处理普通的文件，SequceFileInputFormat用来处理Sequce格式文件。
FileInputFormat类中的getSplits(JobContext job)方法是划分split的主要逻辑。

对于InputSplit，其也是一个抽象类，有多个不同实现。需要注意的是，不同的InputFormat划分出来的InputSplit也不一样，如下图
![这里写图片描述](https://images2015.cnblogs.com/blog/760432/201510/760432-20151027110357669-1939788266.png)

对于FileInputFormat，划分出来的是FileSplit.对于FileSplit，比较重要的如下
```java
public class FileSplit extends InputSplit implements Writable {
  private Path file; //hdfs文件地址
  private long start; //该split的起始位置
  private long length; //该split的文件长度
```



# 3、怎么划分split
划分的主要逻辑是FileInputFormat类中的getSplits(JobContext job)

## 3.1、计算splitsize
划分split的时候会根据splitsize将大文件切分。
涉及到的java代码如下：
```java
   long minSize = Math.max(getFormatMinSplitSize(), getMinSplitSize(job));
   long maxSize = getMaxSplitSize(job);
   long blockSize = file.getBlockSize();
   long splitSize = computeSplitSize(blockSize, minSize, maxSize);
   protected long computeSplitSize(long blockSize, long minSize,
                                  long maxSize) {
    return Math.max(minSize, Math.min(maxSize, blockSize));
  }
```
- minSize :每个split的最小值，默认为1.getFormatMinSplitSize()为代码中写死，固定返回1，除非修改了hadoop的源代码.getMinSplitSize(job)取决于参数mapreduce.input.fileinputformat.split.minsize，如果没有设置该参数，返回1.故minSize默认为1.

-maxSize：每个split的最大值，如果设置了mapreduce.input.fileinputformat.split.maxsize，则为该值，否则为Long的最大值。

-blockSize ：默认为HDFS设置的文件存储BLOCK大小。注意：该值并不一定是唯一固定不变的。HDFS上不同的文件该值可能不同。故将文件划分成split的时候，对于每个不同的文件，需要获取该文件的blocksize。

-splitSize ：根据公式，默认为blockSize 。


>
在本例子中，mapreduce.input.fileinputformat.split.maxsize=104857440 （100M），所有文件的blockSize 大小都是256M，故splitSize=100M

## 3.2、 划分split

划分的逻辑如下：

- 1） 遍历输入目录中的每个文件，拿到该文件
- 2）计算文件长度，A:如果文件长度为0，如果```mapred.split.zero.file.skip=true```，则不划分split ; 如果```mapred.split.zero.file.skip```为false，生成一个length=0的split .B:如果长度不为0，跳到步骤3
- 3）判断该文件是否支持split :如果支持，跳到步骤4;如果不支持，该文件不切分，生成1个split，split的length等于文件长度。
- 4）根据当前文件，计算```splitSize```，本文中为100M
- 5 ) 判断```剩余待切分文件大小/splitsize```是否大于```SPLIT_SLOP```(该值为1.1，代码中写死了) 如果true，切分成一个split，待切分文件大小更新为当前值-splitsize ，再次切分。生成的split的length等于splitsize； 如果false 将剩余的切到一个split里，生成的split length等于剩余待切分的文件大小。之所以需要判断```剩余待切分文件大小/splitsize```,主要是为了避免过多的小的split。比如文件中有100个109M大小的文件，如果```splitSize```=100M，如果不判断```剩余待切分文件大小/splitsize```，将会生成200个split，其中100个split的size为100M，而其中100个只有9M，存在100个过小的split。MapReduce首选的是处理大文件，过多的小split会影响性能。


## 3.3、 实例分析

文中讲解用到的输入文件如下:

>
[test@dw01 ~]$ hadoop fs -dus -h hdfs://namenode.test.net:9000/tmp/wordcount/input/*
hdfs://namenode.test.net:9000/tmp/wordcount/input/part-r-00000       246.32M
hdfs://namenode.test.net:9000/tmp/wordcount/input/part-r-00002       106.95M
hdfs://namenode.test.net:9000/tmp/wordcount/input/part-r-00003       7.09M
hdfs://namenode.test.net:9000/tmp/wordcount/input/part-r-0004        0

在本例子中，```mapreduce.input.fileinputformat.split.maxsize=104857440 （100M）```，```mapred.split.zero.file.skip=true```，所有文件的```blockSize``` 大小都是256M，故```splitSize```=100M

- hdfs://namenode.test.net:9000/tmp/wordcount/input/part-r-00000 划分成3个split
- hdfs://namenode.test.net:9000/tmp/wordcount/input/part-r-00002 划分成1个split
- hdfs://namenode.test.net:9000/tmp/wordcount/input/part-r-00003 划分成1个split
- hdfs://namenode.test.net:9000/tmp/wordcount/input/part-r-0004  不划分split 