---
layout: post
comments: true
date: 2016-05-17
categories: Tez
tags: hadoop Tez oozie
---

* content
{:toc}


## 1.完成上一篇的基础的相关配置

## 2.拷贝Tez的依赖Jar包到OOZIE的HDFS共享目录下
```
hadoop fs -copyFromLocal *.jar /user/oozie/share/lib/lib_20150722203343/hive/
hadoop fs -copyFromLocal /usr/lib/tez/lib/*.jar /user/oozie/share/lib/lib_20150722203343/hive/
```

## 3.修改Jar的权限
保证oozie有权限读取、使用Jar包:
```
hadoop fs -chown oozie:oozie /user/oozie/share/lib/lib_20150722203343/hive/*.jar
```

## 4.是配置生效
```
oozie admin -sharelibupdate
oozie admin -shareliblist hive
```
或者重启oozie也可以

## 5.在workflow里使用Tez
这里咱们只是让oozie处理hive作业时使用Tez引擎，具体配置如下.

### 5.1.使单个workflow里的hive都用tez
在作业流的hive-site.xml中加入下面的配置，即可使整个作业里的hive都使用Tez引擎：

```
<property>
	<name>hive.execution.engine</name>
	<value>tez</value>
</property>
<property>
	<name>tez.lib.uris</name>
	<value>${nameNode}/tmp/apps/tez-0.8.2/,${nameNode}/tmp/apps/tez-0.8.2/lib</value>
</property>
<property>
	<name>tez.use.cluster.hadoop-libs</name>
	<value>true</value>
</property>
```

### 5.2.使单个workflow里的单个hive都用tez
上面的配置不用加，在workflow.xml里的hive节点添加如下配置:

```
<property>
	<name>hive.execution.engine</name>
	<value>tez</value>
</property>
<property>
	<name>tez.lib.uris</name>
	<value>${nameNode}/tmp/apps/tez-0.8.2/,${nameNode}/tmp/apps/tez-0.8.2/lib</value>
</property>
<property>
	<name>tez.use.cluster.hadoop-libs</name>
	<value>true</value>
</property>
```

hive.execution.engine属性可以不添加，在hive的脚本中的第一行添加:
```
set hive.execution.engine=tez;
```
