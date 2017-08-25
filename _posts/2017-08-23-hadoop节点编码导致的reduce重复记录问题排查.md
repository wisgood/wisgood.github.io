---
layout: post
title:  "linux下调整/tmp目录大小"
categories: Linux
tags:  Linux 运维 lvm 
author: wisgood
---


* content
{:toc}


# 1、背景

组内一个同学反馈:reduce输出目录中竟然出现了2条重复的key，理论上同一个key只会有一条记录。程序是通过mr跑的，代码如下：
 ![这里写图片描述](http://img.blog.csdn.net/20170825153805090?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2lzZ29vZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
mapreduce的逻辑很简单，其实就是实现一个去重。原因是我们的上游日志里经常会有重复记录。为了保证结果正确，需要将重复记录去掉。
该同学反馈的这个case中，输入文件中有2条重复记录，且在2个不同文件中。

# 2、问题排查

## 2.1 判断是不是不可见字符

首先怀疑该同学眼神不好，毕竟人眼还没进化到可以识别不可见字符。高度怀疑这2条记录里面有不可见字符，看着是一样的，但其实不一样。
```  sql
select md5(line) ,line
from  temp_test
where line like '%0001a794d86f0844 %' ;
 ```
 ![这里写图片描述](http://img.blog.csdn.net/20170825153823208?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2lzZ29vZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
结果发现，两条记录md5值是一样的。好吧，错怪这名同学了。继续往下排查。
然后怀疑是partition过程出了问题。如果2个key相同的记录，partition过程中分到的不同的reduce，那最终的输出结果里是有可能出现重复记录的。

## 2.2 初步判断partition过程中有问题

理论上，如果两个key一样的话，partition过程会分到同一个reduce里，2条记录应该在同一个reduce文件里（注：每个reduce的输出都在其编号所对应的part-r-编号文件里）。
然后查找” 0001a794d86f0844”这条记录所在的文件位置，结果见下图，最后一列是文件所在位置。奇怪的是key相同的2条记录被分到2个不同的reduce。据此可以初步判断，partition过程中出现了什么问题，导致同一个key分到了不同的reduce。![这里写图片描述](http://img.blog.csdn.net/20170825153849001?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2lzZ29vZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


## 2.3 partition出错原因分析

同一个key分到不同的reduce，首先第1种可能：自己重写了partition函数，写low了。跟该同学了解，该同学还没牛逼到自定义Partition类的程度，故排除该可能 。
然后看了下hadoop的源码，Hadoop中默认的Partition类是HashPartitioner，代码如下。逻辑也比较简单。没看出啥问题。
![这里写图片描述](http://img.blog.csdn.net/20170825153906845?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2lzZ29vZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

看了一会，没看出是啥原因。这个时候公司的下午茶来了：香蕉。吃完香蕉后，满血复活。接着又泡了个茶叶，继续盯代码。古人云：“读书百遍，其义自见”。
突然想起来，处理的日志当中有中文。于是联想到 ：会不会是因为hadoop节点的字符编码不一致，导致不同split的同一个key ，hashcode返回值不一致，计算得到的partition不一样。决定验证一下。

## 2.4 partition出错验证

验证的步骤如下：
1、	修改代码，修改Mapper类。如下。
1）	mapper类里初始化一个HashPartitioner
2）	修改map的输出，将parttition编号一块输出一下。这里默认reduce个数为600。
3）	同时把机器环境打印一下。
 ![这里写图片描述](http://img.blog.csdn.net/20170825153916899?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2lzZ29vZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

2、	修改reduce个数为0。本场景只需要验证map阶段的输出，不需要用到reduce，故需要将reduce个数设置为0。
按照上述，修改完之后，重跑程序。然后果然在输出记录里发现（如下图），这两条一样的记录，分区数一个是373，一个是560.
 ![这里写图片描述](http://img.blog.csdn.net/20170825153955811?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2lzZ29vZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
同时也能够找到对应的hdfs文件。由于hdfs文件的编号和map task的编号一致，根据文件名我们推测，map中07635和07634的机器环境可能不一致。找打对应task，查看日志（因为我们的mapper类里把机器的编码配置打印到了日志里）。

m_007635的输出：
![这里写图片描述](http://img.blog.csdn.net/20170825153935885?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2lzZ29vZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

而m_007634的输出如下。明显看到两台节点上的编码配置是不一样的。问题最终定位。
![这里写图片描述](http://img.blog.csdn.net/20170825153943198?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd2lzZ29vZA==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

# 3、原因分析及解决方案
可以通过下面方法。

## 3.1 原因分析

由于hadoop节点编码不一致，导致在2个节点上执行的同一条记录，获取到了不同的partition，被分到了不同的reduce里，导致了重复记录。

## 3.2 解决方案

1种解决方案是要求系统部把节点编码配置搞成一致。
另1种解决方案是hadoop任务设置参数，加
 ```
 –D mapred.child.env="LANG=en_US.UTF-8,LC_ALL=en_US.UTF-8"
 ```
  这个选项，设置任务执行时的编码。
