---
layout: post
title:  "使用MySQL实现分布式资源调度"
date:   2015-12-25 21:00:56
categories: 分布式
tags: MySQL 分布式
---

使用Hbase来存储Graphx的节点与边，是一种很棒的实践。

### 一、简介

Hbase是当下流行的宽表数据库，基于Hadoop构建。它提供了对百万级列、千亿级行大宽表的支持。不过，HBase的Rowkey、ColumnFamily、Column等概念让人非常困惑，HBase-client逗比的编程模型也令人非常恼火。

Graphx是Apache Spark的图计算组件，提供基于SparkRDD的简洁图计算抽象。

使用Hbase来存储Graphx的节点与边，是一种很棒的实践。

### 二、引入依赖

在build.sbt 中 引入Spark核心库。

``` scala
libraryDependencies += "org.apache.spark" % "spark-sql_2.10" % "1.4.1" % "provided"

libraryDependencies += "org.apache.spark" % "spark-graphx_2.10" % "1.4.1" % "provided"

libraryDependencies += "org.apache.spark" % "spark-hive_2.10" % "1.4.1"

libraryDependencies += "org.apache.spark" % "spark-mllib_2.10" % "1.4.1" % "provided"

libraryDependencies += "org.apache.spark" % "spark-streaming_2.10" % "1.4.1" % "provided"

libraryDependencies += "org.apache.spark" % "spark-streaming-kafka_2.10" % "1.4.1"

libraryDependencies += "org.apache.spark" % "spark-yarn_2.10" % "1.4.1"
```

在Spark的工程中引入HBase的时候，由于HBase的Client库依赖于Junit4.10，而Junit4.10的依赖harmcrest似乎有问题，所以笔者在引入hbase-client的时候，将Junit排除掉了。


``` scala

libraryDependencies += "org.apache.hbase"  % "hbase-common"  % hbaseVersion % "compile" excludeAll ExclusionRule(organization = "junit")

libraryDependencies += "org.apache.hbase"  % "hbase-client"  % hbaseVersion % "compile" excludeAll ExclusionRule(organization = "junit")

```
### 三、编写Hbase Connector

配置好了sbt之后，我们来写一个Hbase Connector。

``` scala
import org.apache.hadoop.hbase.HBaseConfiguration
import org.apache.hadoop.hbase.client.{Put, HTable}
import org.apache.hadoop.hbase.util.Bytes
import scala.collection.JavaConversions._

/**
 * Author   : xiaogo
 * Date     : 2016-01-11 10:48
 * Company  : SZ Zbmy Co.,Ltd
 */

class HBaseConnector (table:String){
    val config = HBaseConfiguration.create()
    val tbl = new HTable(config,table)

    def write(family:String,column :String ,rowkey:String , value:String) = {
        val p = new Put(Bytes.toBytes(rowkey)).add(Bytes.toBytes(family),Bytes.toBytes(column),Bytes.toBytes(value))
        tbl.put(p)
        tbl.flushCommits()
    }

    def batchWrite(family:String,column :String, kvpair:Traversable[(String,String)]) = {
        val lis: List[Put] = kvpair.map(x =>
            new Put(Bytes.toBytes(x._1)).add(Bytes.toBytes(family), Bytes.toBytes(column), Bytes.toBytes(x._2))) .toList
        print("try batchWrite to table "+table + ", count : " + lis.size)
        tbl.put(lis)
        tbl.flushCommits()
    }

    def close() = {
        this.tbl.close()
    }

    def batchMode() = {
        tbl.setAutoFlush(false,true)
        this
    }
}

```

四、编写Spark job

``` scala

import HBaseConnector
import org.apache.spark.graphx.{Edge, Graph, VertexId}
import org.apache.spark.storage.StorageLevel
import org.apache.spark.{SparkContext, SparkConf}
import org.apache.spark.sql.hive.HiveContext


/*  从边中构造graph*/
val graph = Graph.fromEdges(edges,"Default",
    vertexStorageLevel = StorageLevel.MEMORY_AND_DISK_SER_2,
    edgeStorageLevel = StorageLevel.MEMORY_AND_DISK_SER_2)

val Ord = new Ordering[(VertexId,Double)]{
    override def compare(x: (VertexId, Double), y: (VertexId, Double)): Int = x._2.compareTo(y._2)
}

/* 运行PageRank */
val pr = graph.pageRank(0.1)
pr.vertices.
    foreachPartition( x => {
        val hbaseClient = new HBaseConnector("people").batchMode()
        /* 保存顶点信息， 边同理*/
        hbaseClient.batchWrite("power","value",x.map(x=>(long2str(x._1),x._2.toString)).toSeq)
        hbaseClient.close()
    }
)
```

