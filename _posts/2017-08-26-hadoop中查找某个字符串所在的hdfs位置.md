---
layout: post
title:  "hadoop中查找某个字符串所在的hdfs位置"
categories: Hadoop
tags:  Hadoop
author: wisgood
---


* content
{:toc}


在/home/test/2017-08-23这个目录中查找包含0001a794d86f0844的文件

# 1、shell for循环

适用于hdfs容量比较小的的
```shell
for file in `hadoop fs -ls /home/test/2017-08-23 |awk '{print $NF}'`
do echo $file
hadoop fs -text $file |fgrep "0001a794d86f0844" --color
done
```


# 2、借助hive中的虚拟列

适用于查找比较大的文件，借助于mr

创建临时表，只有一列。如：
```sql
create external table temp_find_table (line string )
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\\U0000'
location '/home/test/2017-08-23' ;

select line,INPUT__FILE__NAME ,BLOCK__OFFSET__INSIDE__FILE
from  temp_find_table
where line like '%0001a794d86f0844%' ;

```

其中第2列是记录所在文件位置，第3列是记录在文件中的偏移量。
