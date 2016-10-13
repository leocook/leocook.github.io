---
layout: post
comments: true
date: 2016-10-12
categories: spark
tags: spark
---

* content
{:toc}

Spark streaming是基于Spark core的，天然的具备易扩展、高吞吐量，以及自动容错等特性。支持的主流数据源有Kafka、Flume、HDFS、Twitter、TCP socket等等，Spark的数据输出在spark streaming中都支持，对于有spark基础的开发人员来说，开发spark streaming应用成本将会少很多。本文是一篇概述型的文章，相关详细的配置会在后边逐渐补上。

Spark streaming在企业实战中经常回到这么几类问题：内存溢出、外部系统连接数过多、分配的资源超过了程序所需要的资源，造成资源浪费。



## 1.内存溢出

Spark内存大概分为了两类：Execution Memory和Storage Memory，前者主要用来做buffer的，例如joins、shuffle、sort等等；后者主要用来做cache，例如RDD的数据存储、广播变量、task结果数据等等。从1.6版本之前这两部分内存是不能共享的，从1.6开始之后这两部分内存就可以共享了。

我遇到的内存溢出问题大概有三类，通常是单个分区处理的数据量过多：

- a).数据倾斜引；   
- b).数据未倾斜，分区数过少；   
- c).某个分区中产生了一个较大的内存集合，例如大的List、Set，或者Map。   

这类问题在spark core中也是经常会出现的，关于数据倾斜的问题以后会详细讲解。入手一个新的业务时，应该对该业务的数据量有个大概的预估，这样给这个应用分配多少节点，每个节点会处理多大的数据量，这样就能很好的预估出每个节点分配多少资源比较合理了。

为了避免内存溢出，可以从下面几个方向来做：
- a).避免数据倾斜；
- b).评估每个分区的数据量，给每个分区分配合理的资源；
- c).控制好List、Set等集合的大小；
- d).控制Spark streaming读取源数据的最大速度（spark.streaming.kafka.maxRatePerPartition），实时数据流量有高峰和低谷，不同时间处理的数据量是不一样的，为防止数据高峰的时候内存溢出，这里有必要做配置；
- e).配置Spark streaming可动态控制读取数据源的速度（spark.streaming.backpressure.enabled）。


## 2.外部系统连接数过多

在使用spark streaming解决问题的时候，经常会对外部数据源进行读写操作，切记每次读写完成之后需要关闭网络连接。在我们以往的编程经验中，这些思维都是一直持有的。   
在往hbase写入数据的时候，如果你使用了RDD.saveAsHadoopDataset方法，就需要注意了，org.apache.hadoop.hbase.mapred.TableOutputFormat类存在bug：不能释放zookeeper连接，导致在往hbase写数据的时候，zookeeper的连接数不停得增长。

推荐使用下面这种写法：

```

import org.apache.hadoop.mapreduce.Job
import org.apache.hadoop.hbase.mapreduce.TableOutputFormat

val conf = HBaseConfiguration.create()
conf.set("hbase.zookeeper.quorum", zk_hosts)
conf.set("hbase.zookeeper.property.clientPort", zk_port)

conf.set(TableOutputFormat.OUTPUT_TABLE, "TABLE_NAME")
val job = Job.getInstance(conf)
job.setOutputFormatClass(classOf[TableOutputFormat[String]])

formatedLines.map{
  case (a,b, c) => {
    val row = Bytes.toBytes(a)

    val put = new Put(row)
    put.setDurability(Durability.SKIP_WAL)

    put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("node"), Bytes.toBytes(b))
    put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("topic"), Bytes.toBytes(c))

    (new ImmutableBytesWritable(row), put)
  }
}.saveAsNewAPIHadoopDataset(job.getConfiguration)

```

## 3.资源分配的浪费

如果实时数据在每天的某个时间点有着平时的几倍的数据量，如果给该作业分配过多的资源，那么在绝大多数，这些资源都是闲置浪费的。这里可以启用动态资源分配。

关于配置介绍可以查看官方文档：http://spark.apache.org/docs/latest/job-scheduling.html#dynamic-resource-allocation

如果启用该配置，需要做如下配置：

>1.在spark应用中配置spark.dynamicAllocation.enabled=true   
2.每个节点启动外部shuffle服务，并在spark应用中配置spark.shuffle.service.enabled=true

关于外部shuffle服务，在standalone、Mesos，yarn中的配置是不一样的。

- standalone

启动worker的时候指定spark.shuffle.service.enabled=true

- Mesos

在所有节点上配置spark.shuffle.service.enabled=true，然后执行$SPARK_HOME/sbin/start-mesos-shuffle-service.sh

- yarn

a.添加yarn的配置文件，重新编译spark。如果使用官方编译好的安装包，可以忽略这一步。
b.找到spark-<version>-yarn-shuffle.jar。如果自己编译spark的话，在目录$SPARK_HOME/common/network-yarn/target/scala-<version>下；如果是使用官方编译好的spark，在lib目录下寻找。
c.添加到spark-<version>-yarn-shuffle.jar到yarn所有NodeManager的classpath下。
d.配置所有NodeManager的yarn-site.xml文件如下：

```

<property>
  <name>yarn.nodemanager.aux-services</name>
  <value>spark_shuffle</value>
</property>
<property>
  <name>yarn.nodemanager.aux-services.spark_shuffle.class</name>
  <value>org.apache.spark.network.yarn.YarnShuffleService</value>
</property>

```

e.重启所有的NodeManager

spark on yarn配置了外部shuffle之后，<code>--num-executors</code>配置将不再生效。







