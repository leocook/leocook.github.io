---
layout: post
comments: true
date: 2016-05-09
categories: tez
tags: hadoop tez hive
---

* content
{:toc}

本文主要描述Tez的安装配置，以及使用Tez作为Hive的计算引擎时的相关配置。

## 1.安装配置Tez

### 1.1.环境要求

- CDH5.4.4(hadoop2.6.0)
- 编译环境：gcc, gcc-c++, make, build
- Nodejs、npm (Tez-ui需要)
- Git
- pb2.5.0
- maven3
- Tez0.8.2





### 1.2.集群准备
以及安装完成的cdh5.4.4集群。

### 1.3. 编译环境准备
安装gcc, gcc-c++, make, build    
```
yum install gcc gcc-c++ libstdc++-devel make build
```

### 1.4. Nodejs、npm
下载源码:    
```
wget http://nodejs.org/dist/v0.8.14/node-v0.8.14.tar.gz
```    

解压后编译:    
```
./configure
make && make install
```

查看nodejs和npm的版本:    
```
node --version
npm --version
```    

笔者的安装环境里，node的版本是v0.12.9，npm的版本是2.14.9


### 1.5.安装GIT
下载git:    
```
https://git-scm.com/download    
笔者选择的是1.7.3版本
```

解压后编译:    
```
./configure
make
make install
```

### 1.6. ProtocolBuffer2.5.0
- 下载pb源码    
```
https://github.com/google/protobuf/releases/tag/v2.5.0
```
- 编译安装pb    
```
./configure -prefix=/opt/protoc/
make && make install
```

- 配置环境变量   
```
export PROTOC_HOME=/opt/protoc
export PATH=$PATH:$PROTOC_HOME/bin
```
- 检验    
```
protoc --version
```    
查看pb的版本是不是2.5.0，笔者这里显示为 **libprotoc 2.5.0**

### 1.7.编译&安装Tez
- 下载Tez   
Tez所有版本列表在者：

```
http://tez.apache.org/releases/index.html
```
笔者这里下载的是0.8.2版本。

- 解压修改配置    

vi pom.xml    

```
<hadoop.version>2.6.0-cdh5.4.4</hadoop.version>
```

 
vi tez-ui/pom.xml    

```
<nodeVersion>v0.12.9</nodeVersion>
<npmVersion>2.14.9</npmVersion>
```

- 开始编译    

```
mvn clean package -DskipTests=true -Dmaven.javadoc.skip=true  -Dfrontend-maven-plugin.version=0.0.23
```

>编译的过程中可能会发生错误，我这边由于网络故障，经常会出现node.gz.tar文件下载失败。最后还是编译成功了。


## 2. 开始整合Hive和Tez

### 2.1. 查看编译完成的目标目录结构
```
[wulin@lf-R710-29 target]$ ls
archive-tmp  maven-archiver  tez-0.8.2  tez-0.8.2-minimal  tez-0.8.2-minimal.tar.gz  tez-0.8.2.tar.gz  tez-dist-0.8.2-tests.jar
```

### 2.1. 拷贝tez-0.8.2-minimal目录至HDFS
```
hdfs dfs -put tez-0.8.2-minimal /tmp/tez-dir/
```
先拷贝到tmp目录做测试，成功运行后在拷贝到正式目录。

### 2.2. 拷贝对应以来jar
把hadoop-mapreduce-client-common-2.6.0-cdh5.4.4.jar到hdfs的/tmp/tez-dir/tez-0.8.2-minimal目录

### 2.3. 把tez-0.8.2拷贝到服务器本地部署的目录
```
cp -r tez-0.8.2 /opt/
```

### 2.4. 进入部署的目录创建conf/tez-site.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
 <property>
   <name>tez.lib.uris</name>
   <value>${fs.defaultFS}/tmp/tez-dir/tez-0.8.2-minimal,${fs.defaultFS}/tmp/tez-dir/tez-0.8.2-minimal/lib</value>
 </property>
 <property>
   <name>tez.use.cluster.hadoop-libs</name>
   <value>true</value>
 </property>
</configuration>
```
>注：tez.lib.uris参数值，是之前上传到hdfs目录的tez包，它必须是tez-0.8.2-minimal目录，而不能是tez-0.8.2目录。（如果有谁使用tez-0.8.2目录部署成功的话，可以告诉我，谢谢！）。根据官网的说明，使用tez-0.8.2-minimal包的时候，务必设置tez.use.cluster.hadoop-libs属性为true。

### 2.4. 把Tez加入到环境变量

```
export TEZ_JARS=/opt/tez-0.8.2-minimal
export TEZ_CONF_DIR=$TEZ_JARS/conf
export HADOOP_CLASSPATH=${TEZ_CONF_DIR}:${TEZ_JARS}/*:${TEZ_JARS}/lib/*
```
>注：经笔者的测试，TEZ_JARS指向tez-0.8.2-minimal目录或者tez-0.8.2目录都是可以的。

### 2.5. 让Hive把Tez用起来
- 配置整合
临时配置

```
hive>set hive.execution.engine=tez;
```
或者修改hive-site.xml（长期配置）

```
 <property>
   <name>hive.execution.engine</name>
   <value>tez</value>
 </property>
```
- 执行hive，验证查看

```
hive> select count(*) from dual;
Query ID = wulin_20160406152121_6fd704e7-a437-4345-9958-2fbd1cccb057
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks determined at compile time: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
Starting Job = job_1457012272029_352465, Tracking URL = http://lfh-R710-165:8088/proxy/application_1457012272029_352465/
Kill Command = /opt/cloudera/parcels/CDH-5.4.4-1.cdh5.4.4.p0.4/lib/hadoop/bin/hadoop job  -kill job_1457012272029_352465
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 1
2016-04-06 15:21:29,925 Stage-1 map = 0%,  reduce = 0%
2016-04-06 15:21:38,274 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 2.05 sec
2016-04-06 15:21:45,611 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 3.86 sec
MapReduce Total cumulative CPU time: 3 seconds 860 msec
Ended Job = job_1457012272029_352465
MapReduce Jobs Launched: 
Stage-Stage-1: Map: 1  Reduce: 1   Cumulative CPU: 3.86 sec   HDFS Read: 6138 HDFS Write: 2 SUCCESS
Total MapReduce CPU Time Spent: 3 seconds 860 msec
OK
1
Time taken: 35.934 seconds, Fetched: 1 row(s)
hive> set hive.execution.engine=tez;
hive> select count(*) from dual;
Query ID = wulin_20160406152222_426dd505-1f6a-4d02-ae95-5a4d0e6bbc76
Total jobs = 1
Launching Job 1 out of 1
Status: Running (Executing on YARN cluster with App id application_1457012272029_352467)
--------------------------------------------------------------------------------
        VERTICES      STATUS  TOTAL  COMPLETED  RUNNING  PENDING  FAILED  KILLED
--------------------------------------------------------------------------------
Map 1 ..........   SUCCEEDED      1          1        0        0       0       0
Reducer 2 ......   SUCCEEDED      1          1        0        0       0       0
--------------------------------------------------------------------------------
VERTICES: 02/02  [==========================>>] 100%  ELAPSED TIME: 9.64 s     
--------------------------------------------------------------------------------
OK
1
Time taken: 22.211 seconds, Fetched: 1 row(s)
hive> 
```
如下图：
![Hive on Tez](http://7xriy2.com1.z0.glb.clouddn.com/tez-ok.png "Hive on Tez")

到此，hive on tez，整合完毕！

## 常见的错误

### 1.不要使用sudo权限来编译（编译tez时的错误）
```
[ERROR] Failed to execute goal org.codehaus.mojo:exec-maven-plugin:1.3.2:exec (Bower install) on project tez-ui: Command execution failed. Process exited with an error: 1 (Exit value: 1) -> [Help 1]
```
解决办法：
不要使用root用户，也不要使用sudo来编译.

### 2.maven插件frontend-maven-plugin的版本问题（编译tez时的错误）

```
[ERROR] Failed to execute goal com.github.eirslett:frontend-maven-plugin:0.0.22:install-node-and-npm (install node and npm) on project tez-ui: Execution install node and npm of goal com.github.eirslett:frontend-maven-plugin:0.0.22:install-node-and-npm failed: A required class was missing while executing com.github.eirslett:frontend-maven-plugin:0.0.22:install-node-and-npm: org/slf4j/helpers/MarkerIgnoringBase
```
解决办法：强制执行编译时frontend-maven-plugin插件的版本（mvn clean package -DskipTests=true -Dmaven.javadoc.skip=true  -Dfrontend-maven-plugin.version=0.0.XX）
如果maven的版本低于3.1，frontend-maven-plugin版本应该 <= 0.0.22；
如果maven的版本大于或等于3.1，frontend-maven-plugin版本应该 >= 0.0.23.

### 3.解压Node压缩包时错误（编译tez时的错误）

```
[ERROR] Failed to execute goal com.github.eirslett:frontend-maven-plugin:0.0.22:install-node-and-npm (install node and npm) on project tez-ui: Could not extract the Node archive: Could not extract archive: '/home/.../tez/tez-ui/src/main/webapp/node_tmp/node.tar.gz': EOFException -> [Help 1]
```
解决版本：检查第二个问题，并重新执行。如果还失败，可以多执行几次，可能和网络有关系。

### 4.node&npm版本不对应（编译tez时的错误）

```
[ERROR] npm WARN engine hoek@2.16.3: wanted: {"node":">=0.10.40"} (current: {"node":"v0.10.18","npm":"1.3.8"})
[ERROR] npm WARN engine boom@2.10.1: wanted: {"node":">=0.10.40"} (current: {"node":"v0.10.18","npm":"1.3.8"})
[ERROR] npm WARN engine cryptiles@2.0.5: wanted: {"node":">=0.10.40"} (current: {"node":"v0.10.18","npm":"1.3.8"})
```
解决办法：
1. 安装正确版本的nodeJs；
2. 修改tez-ui/pom.xml中的nodeVersion和npmVersion标签值为系统环境的值。可使用下面命令，查看系统里的node和npm版本：
```
node --version
npm --version
```

### 5.缺少MR的依赖包（error when run hive on tez）
```
Vertex failed, vertexName=Map 1, vertexId=vertex_1457012272029_352429_1_00, diagnostics=[Vertex vertex_1457012272029_352429_1_00 [Map 1] killed/failed due to:ROOT_INPUT_INIT_FAILURE, Vertex Input: dual initializer failed, vertex=vertex_1457012272029_352429_1_00 [Map 1], java.lang.NoClassDefFoundError: org/apache/hadoop/mapred/MRVersion
        at org.apache.hadoop.hive.shims.Hadoop23Shims.isMR2(Hadoop23Shims.java:843)
        at org.apache.hadoop.hive.shims.Hadoop23Shims.getHadoopConfNames(Hadoop23Shims.java:914)
        at org.apache.hadoop.hive.conf.HiveConf$ConfVars.<clinit>(HiveConf.java:356)
        at org.apache.hadoop.hive.ql.exec.Utilities.getBaseWork(Utilities.java:371)
        at org.apache.hadoop.hive.ql.exec.Utilities.getMapWork(Utilities.java:296)
        at org.apache.hadoop.hive.ql.exec.tez.HiveSplitGenerator.initialize(HiveSplitGenerator.java:106)
        at org.apache.tez.dag.app.dag.RootInputInitializerManager$InputInitializerCallable$1.run(RootInputInitializerManager.java:278)
        at org.apache.tez.dag.app.dag.RootInputInitializerManager$InputInitializerCallable$1.run(RootInputInitializerManager.java:269)
        at java.security.AccessController.doPrivileged(Native Method)
        at javax.security.auth.Subject.doAs(Subject.java:415)
        at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1628)
        at org.apache.tez.dag.app.dag.RootInputInitializerManager$InputInitializerCallable.call(RootInputInitializerManager.java:269)
        at org.apache.tez.dag.app.dag.RootInputInitializerManager$InputInitializerCallable.call(RootInputInitializerManager.java:253)
        at java.util.concurrent.FutureTask.run(FutureTask.java:262)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
        at java.lang.Thread.run(Thread.java:745)
Caused by: java.lang.ClassNotFoundException: org.apache.hadoop.mapred.MRVersion
        at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
        at java.security.AccessController.doPrivileged(Native Method)
        at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
        at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:308)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
        ... 17 more
]
Vertex killed, vertexName=Reducer 2, vertexId=vertex_1457012272029_352429_1_01, diagnostics=[Vertex received Kill in INITED state., Vertex vertex_1457012272029_352429_1_01 [Reducer 2] killed/failed due to:OTHER_VERTEX_FAILURE]
DAG did not succeed due to VERTEX_FAILURE. failedVertices:1 killedVertices:1
FAILED: Execution Error, return code 2 from org.apache.hadoop.hive.ql.exec.tez.TezTask
```
这个错误是在配置完之后，运行hive时才会出现的。
解决办法：
拷贝mr依赖包至tez的hdfs目录中。笔者的环境是CDH5.4.4，所以把hadoop-mapreduce-client-common-2.6.0-cdh5.4.4.jar拷贝到hdfs的/tmp/tez-dir/tez-0.8.2-minimal目录，就解决问题了。

### 6.tez&hive on oozie 错误

```
Status: Running (Executing on YARN cluster with App id application_1461470184587_0770)

Map 1: -/-	Reducer 2: 0/1	
Status: Failed
Vertex failed, vertexName=Map 1, vertexId=vertex_1461470184587_0770_1_00, diagnostics=[Vertex vertex_1461470184587_0770_1_00 [Map 1] killed/failed due to:ROOT_INPUT_INIT_FAILURE, Vertex Input: wl_manager_core_assembly initializer failed, vertex=vertex_1461470184587_0770_1_00 [Map 1], java.lang.IllegalArgumentException: Illegal Capacity: -1
	at java.util.ArrayList.<init>(ArrayList.java:142)
	at org.apache.hadoop.mapred.FileInputFormat.getSplits(FileInputFormat.java:330)
	at org.apache.hadoop.hive.ql.io.HiveInputFormat.addSplitsForGroup(HiveInputFormat.java:306)
	at org.apache.hadoop.hive.ql.io.HiveInputFormat.getSplits(HiveInputFormat.java:408)
	at org.apache.hadoop.hive.ql.exec.tez.HiveSplitGenerator.initialize(HiveSplitGenerator.java:129)
	at org.apache.tez.dag.app.dag.RootInputInitializerManager$InputInitializerCallable$1.run(RootInputInitializerManager.java:278)
	at org.apache.tez.dag.app.dag.RootInputInitializerManager$InputInitializerCallable$1.run(RootInputInitializerManager.java:269)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:415)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1671)
	at org.apache.tez.dag.app.dag.RootInputInitializerManager$InputInitializerCallable.call(RootInputInitializerManager.java:269)
	at org.apache.tez.dag.app.dag.RootInputInitializerManager$InputInitializerCallable.call(RootInputInitializerManager.java:253)
	at java.util.concurrent.FutureTask.run(FutureTask.java:262)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
	at java.lang.Thread.run(Thread.java:745)
]
Vertex killed, vertexName=Reducer 2, vertexId=vertex_1461470184587_0770_1_01, diagnostics=[Vertex received Kill in INITED state., Vertex vertex_1461470184587_0770_1_01 [Reducer 2] killed/failed due to:OTHER_VERTEX_FAILURE]
DAG did not succeed due to VERTEX_FAILURE. failedVertices:1 killedVertices:1
FAILED: Execution Error, return code 2 from org.apache.hadoop.hive.ql.exec.tez.TezTask
Intercepting System.exit(2)
Failing Oozie Launcher, Main class [org.apache.oozie.action.hadoop.HiveMain], exit code [2]
```


参考链接：
http://m.oschina.net/blog/421764    
http://duguyiren3476.iteye.com/blog/2214549
https://cwiki.apache.org/confluence/display/TEZ/Build+errors+and+solutions

