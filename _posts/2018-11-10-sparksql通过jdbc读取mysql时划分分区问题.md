---
layout: post
title:  "sparksql通过jdbc读取mysql时划分分区问题"
categories: Spark
tags:  Spark
author: wisgood
---

* content
{:toc} 


当通过spark读取mysql时，如果数据量比较大，为了加快速度，通常会起多个task并行拉取mysql数据。
其中一个api是
```
def
jdbc(url: String, table: String, columnName: String, lowerBound: Long, upperBound: Long, numPartitions: Int, connectionProperties: Properties): DataFrame
```

| 参数      | 说明         | 
|:-----------| :----------|
| url | 访问mysql时的jdbc链接，如jdbc:mysql://190.1.98.225:2049/test| 
| table| 访问的表| 
| columnName| 用于分区的列，必须是数字类型| 
| lowerBound| 分区列的最小值| 
| upperBound| 分区列的最大值| 
| numPartitions| 预期的分区数| 
| connectionProperties| mysql的配置参数，key value形式| 


这里面容易引起混淆的是lowerBound和upperBound。需要注意的是lowerBound和upperBound仅用于决定划分分区时的步长，而不是用于按照这两个值对数据进行过滤。 因此，无论这两个值如何设置，表中的所有行都将被读取。

同时需要注意的是，尽量不要创建太多分区，否则很容易将mysql搞挂。

关于具体的分区，我写了个示例代码，参考如下（本部分代码参考spark源码org.apache.spark.sql.execution.datasources.jdbc中columnPartition方法 ）。

代码如下：
```java 
import scala.collection.mutable.ArrayBuffer
object PrintJdbcParition {
  case class JDBCPartition(whereClause: String, partitionIndex: Int)
  def main(args: Array[String]): Unit = {
    val numPartitions = 10
    val lowerBound = 100
    val upperBound = 900
    val column = "id"
    // Overflow and silliness can happen if you subtract then divide.
    // Here we get a little roundoff, but that's (hopefully) OK.
    val stride: Long = (upperBound / numPartitions - lowerBound / numPartitions)
    var i: Int = 0
    var currentValue: Long = lowerBound
    var ans = new ArrayBuffer[JDBCPartition]()
    while (i < numPartitions) {
      val lowerBound = if (i != 0) s"$column >= $currentValue" else null
      currentValue += stride
      val upperBound = if (i != numPartitions - 1) s"$column < $currentValue" else null
      val whereClause =
        if (upperBound == null) {
          lowerBound
        } else if (lowerBound == null) {
          upperBound
        } else {
          s"$lowerBound AND $upperBound"
        }
      ans += JDBCPartition(whereClause, i)
      i = i + 1
    }
    ans.toArray.map(println(_))
  }
}
```
代码执行结果如下：

```
JDBCPartition(id < 180,0)
JDBCPartition(id >= 180 AND id < 260,1)
JDBCPartition(id >= 260 AND id < 340,2)
JDBCPartition(id >= 340 AND id < 420,3)
JDBCPartition(id >= 420 AND id < 500,4)
JDBCPartition(id >= 500 AND id < 580,5)
JDBCPartition(id >= 580 AND id < 660,6)
JDBCPartition(id >= 660 AND id < 740,7)
JDBCPartition(id >= 740 AND id < 820,8)
JDBCPartition(id >= 820,9)
```