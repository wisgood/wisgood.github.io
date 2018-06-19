---
layout: post
title:  "mapreduce程序中避免reduce输出空文件"
categories: Hadoop
tags:  Hadoop
author: wisgood
---

* content
{:toc}


在mapreduce里，如果某个reduce输出为空，默认也会生成一个大小为0的文件。原因是reduce写的时候，不知道会不会有输出数据，所以默认初始化了一个文件。如果没有输出，close文件最终会生成一个空文件。如下。有几个缺点：
1）生成的很多小文件，对namenode形成一定压力
2）生成的数据下个阶段处理的时候，这些空的文件会浪费掉一些计算资源。
3）看着不爽

```
-rw-r--r--   3 hadoop supergroup          0 2018-05-09 10:38 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/_SUCCESS
drwxr-xr-x   - hadoop supergroup          0 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/_temporary
-rw-r--r--   3 hadoop supergroup     290779 2018-05-09 10:37 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00000
-rw-r--r--   3 hadoop supergroup     102365 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00001
-rw-r--r--   3 hadoop supergroup     210493 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00002
-rw-r--r--   3 hadoop supergroup     194585 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00003
-rw-r--r--   3 hadoop supergroup      97649 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00004
-rw-r--r--   3 hadoop supergroup          0 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00005
-rw-r--r--   3 hadoop supergroup      74188 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00006
-rw-r--r--   3 hadoop supergroup      61837 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00007
-rw-r--r--   3 hadoop supergroup     254879 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00008
-rw-r--r--   3 hadoop supergroup       6061 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00009
-rw-r--r--   3 hadoop supergroup          0 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00010
-rw-r--r--   3 hadoop supergroup     126900 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00011
-rw-r--r--   3 hadoop supergroup       3623 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00012
-rw-r--r--   3 hadoop supergroup      75816 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00013
-rw-r--r--   3 hadoop supergroup     358310 2018-05-09 10:38 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00014
-rw-r--r--   3 hadoop supergroup     115713 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00015
-rw-r--r--   3 hadoop supergroup      90556 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00016
-rw-r--r--   3 hadoop supergroup     150605 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00017
-rw-r--r--   3 hadoop supergroup          0 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00018
-rw-r--r--   3 hadoop supergroup      54610 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00019
-rw-r--r--   3 hadoop supergroup     163868 2018-05-09 10:36 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00020
```

解决办法很简单，使用```LazyOutputFormat```，从名字能看出，这个是lazy的outputformat。只有真正有数据要输出的时候，才会初始化一个hdfs上的文件，避免了形成空文件。
使用方式如下
```java
//导入jar包
import org.apache.hadoop.mapreduce.lib.output.LazyOutputFormat;
//使用LazyOutputFormat
LazyOutputFormat.setOutputFormatClass(job, TextOutputFormat.class);
```

重跑下数据，现在ok了，空文件都没有了，如下

```
-rw-r--r--   3 hadoop supergroup          0 2018-05-09 11:04 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/_SUCCESS
drwxr-xr-x   - hadoop supergroup          0 2018-05-09 11:03 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/_temporary
-rw-r--r--   3 hadoop supergroup     307566 2018-05-09 11:03 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00000
-rw-r--r--   3 hadoop supergroup     101794 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00001
-rw-r--r--   3 hadoop supergroup     219026 2018-05-09 11:03 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00002
-rw-r--r--   3 hadoop supergroup     195994 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00003
-rw-r--r--   3 hadoop supergroup     101289 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00004
-rw-r--r--   3 hadoop supergroup      73074 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00006
-rw-r--r--   3 hadoop supergroup      60595 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00007
-rw-r--r--   3 hadoop supergroup     258069 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00008
-rw-r--r--   3 hadoop supergroup       6061 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00009
-rw-r--r--   3 hadoop supergroup     127131 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00011
-rw-r--r--   3 hadoop supergroup       3623 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00012
-rw-r--r--   3 hadoop supergroup      75725 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00013
-rw-r--r--   3 hadoop supergroup     366958 2018-05-09 11:04 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00014
-rw-r--r--   3 hadoop supergroup     113457 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00015
-rw-r--r--   3 hadoop supergroup      87421 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00016
-rw-r--r--   3 hadoop supergroup     148302 2018-05-09 11:03 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00017
-rw-r--r--   3 hadoop supergroup      54856 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00019
-rw-r--r--   3 hadoop supergroup     162752 2018-05-09 11:02 /home/hadoop/offline/js_ad/daily_ad_url/2018-05-08/part-r-00020
```