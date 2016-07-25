---
layout: post
comments: true
date: 2016-07-23
categories: hadoop
tags: cm hadoop
---

* content
{:toc}


对于Hadoop这个复杂的大系统，我们期望能有一个平台，可以对Hadoop的每一个部件都能够进行安装部署，以及细颗粒度的监控。Apache发行版的Hadoop可以使用Ambari；Cloudera公司的CDH版本Hadoop则可以使用Cloudera Manager（后面简称为CM）来统一管理和部署。咱们这里的操作系统使用的是ubuntu14.04.




## 1.Cloudera Manager安装前准备

### 1.1. 操作系统的优化
- 打开的最大文件数
修改当前的session配置(临时)：

```
ulimit -SHn 65535
```

永久修改（需要重启服务器）：

```
sudo echo "ulimit -SHn 65535" >> /etc/rc.local
sudo chmod +x /etc/rc.local
```

- 打开的组大文件句柄数
临时配置

```
# 一般可不做修改，这是临时配置
sudo echo 2000000 > /proc/sys/fs/file-max
```

永久配置

```
echo "echo 2000000 > /proc/sys/fs/file-max" >> /etc/rc.local
```

或者


```
sudo echo "fs.file-max = 2000000" >>/etc/sysctl.conf #推荐
#使文件生效sudo /sbin/sysctl -p
```

- 打开的最大网络连接数

```
sudo echo "net.core.somaxconn = 2048" >>/etc/sysctl.conf
sudo /sbin/sysctl -p
```

- 关闭selinux

ubuntu默认是不安装的selinux的，所以这里可以直接忽略。

- 配置ntp

安装

```
apt-get install ntp
```

如果只需要保证集群内部的各个server之间时间保持同步，只需要在需要同步的机器上配置：

```
0 */12 * * * ntpdate dt-vt-154
```

如果需要时间和互联网的时间保持一致，那么就需要在提供ntp server的机器上配置上层ntpserver:

```
server [ntpserver_01]
server [ntpserver_02]
server [ntpserver_03]
```

- 关闭防火墙

```
sudo ufw disable #关闭防火墙
apt-get remove iptables #卸载防火墙
```

- 配置好hosts映射文件

```
vi /etc/hosts

192.168.1.151   dt-vt-151
192.168.1.152   dt-vt-152
192.168.1.153   dt-vt-153
192.168.1.154   dt-vt-154
```

- Java环境配置

安装

```
mkdir /opt/java
cd /opt/java
#下载到安装文件到这个目录
```

环境配置

```
vi /etc/profile

export JAVA_HOME=/opt/java/jdk1.7.0_79
export CLASSPATH=.:$JAVA_HOME/lib
export PATH=$JAVA_HOME/bin:$PATH
```

执行下面命令，使当前的session生效：

```
source /etc/profile
```


### 1.2. 数据的存放
在使用CM来管理集群的时候，会涉及到大量的数据存储，例如Hadoop的主机列表信息，主机的配置信息，负载信息，各个模块的运行时状态等等。
咱们这里使用mysql来作为CM的数据存储，这里不仅是CM，Hadoop中的Hive等模块的元数据，都来使用mysql存储。

- 安装mysql

```
sudo apt-get install mysql-server
```

我这里安装完后是5.5.49.

- 关闭mysql，备份配置文件

```
/etc/init.d/mysql stop
```

把/var/lib/mysql/ib_logfile0和/var/lib/mysql/ib_logfile1拷贝至某个配置目录中，例如：/var/lib/mysql/bak

- 配置InnoDB引擎

务必使用InnoDB引擎引擎，若是使用MyISAM引擎CM将启动不了。在Mysql的命令行中运行下面的命令，来查看你的Mysql使用了哪个引擎。

```
mysql> show table status;
```

- 配置mysql的innodb刷写模式

```
innodb_flush_method=O_DIRECT
```

即：配置Innodb的刷写模式为异步写模式。

- 修改mysql的最大连接数

```
max_connections=1550
```

在这里，你应该会考虑配置该数值为多少比较合适。
当集群规模**小于50台**的时候，假设该库中有N个数据库是用来服务于Hadoop的，那么max_connections可以设置为100*N+50。例如：Cloudera Manager Server, Activity Monitor, Reports Manager, Cloudera Navigator, 和 Hive metastore都是使用mysql的，那么就配置max_connections为550.
当集群规模**大于50台**的时候，建议每个数据库只存放在一台机器上。

- 配置文件样例

```
[mysqld]
transaction-isolation = READ-COMMITTED
# Disabling symbolic-links is recommended to prevent assorted security risks;
# to do so, uncomment this line:
# symbolic-links = 0

key_buffer = 16M
key_buffer_size = 32M
max_allowed_packet = 32M
thread_stack = 256K
thread_cache_size = 64
query_cache_limit = 8M
query_cache_size = 64M
query_cache_type = 1

max_connections = 1550
#expire_logs_days = 10
#max_binlog_size = 100M


# InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit  = 2
innodb_log_buffer_size = 64M
innodb_buffer_pool_size = 4G
innodb_thread_concurrency = 8
innodb_flush_method = O_DIRECT
innodb_log_file_size = 512M

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

```

- 启动mysql

```
/etc/init.d/mysql start
```

打开开机自启

```
apt-get install sysv-rc-conf
sysv-rc-conf mysql on
```

- 安装mysql-jdbc驱动

```
apt-get install libmysql-java
```

- mysql远程连接

```
vi /etc/mysql/my.cnf

bind-address = 0.0.0.0
```

- 給相关服务创建mysql的数据库

相关列表如下：

| Role | Database | User | Password |
| --- | --- | --- | --- |
| Activity Monitor | amon | amon | amon_password |
| Reports Manager | rman | rman | rman_password |
| Hive Metastore Server | hive_metastore | hive | hive_password |
| Sentry Server | sentry | sentry | sentry_password |
| Cloudera Navigator Audit Server | nav | nav | nav_password |
| Cloudera Navigator Metadata Server | navms | navms | navms_password |
| OOZIE | oozie | oozie | oozie |

建表语句格式如下：

```
create database database DEFAULT CHARACTER SET utf8;
grant all on database.* TO 'user'@'%' IDENTIFIED BY 'password';
```

建表语句如下：

```
create database amon DEFAULT CHARACTER SET utf8;
grant all on amon.* TO 'amon'@'%' IDENTIFIED BY 'amon';

create database rmon DEFAULT CHARACTER SET utf8;
grant all on rmon.* TO 'rmon'@'%' IDENTIFIED BY 'rmon';

create database hive_metastore DEFAULT CHARACTER SET utf8;
grant all on hive_metastore.* TO 'hive'@'%' IDENTIFIED BY 'hive';

create database sentry DEFAULT CHARACTER SET utf8;
grant all on sentry.* TO 'sentry'@'%' IDENTIFIED BY 'sentry';

create database nav DEFAULT CHARACTER SET utf8;
grant all on nav.* TO 'nav'@'%' IDENTIFIED BY 'nav';

create database navms DEFAULT CHARACTER SET utf8;
grant all on navms.* TO 'navms'@'%' IDENTIFIED BY 'navms';

create database oozie DEFAULT CHARACTER SET utf8;
grant all on oozie.* TO 'oozie'@'%' IDENTIFIED BY 'oozie';
```

## 2.开始安装Cloudera Manager

### 2.1.下载CM

可以下载安装最新版本的CM：

```
wget http://archive.cloudera.com/cm5/installer/latest/cloudera-manager-installer.bin
```

当然，如果你想选择安装其它版本，可以访问下面的地址，并选择下载你所需要的版本(cm5+)：

```
http://archive.cloudera.com/cm5/installer/
```

我这里使用的是版本是5.4.10，从release note上看，目前cdh5.4.10是最稳当的版本。我下载到了本地路径如下：

```
/opt/cm/cloudera-manager-installer.bin
```

修改权限，使其可以被执行：

```
chmod u+x cloudera-manager-installer.bin
```

### 2.2.配置私有软件仓库(如果不使用私有仓库，这里可以直接跳过)

#### 2.2.1.创建一个临时可以使用的远程仓库
这个配置是在安装cloudera-manager-server的时候才会用的。这里的仓库是使用传统的http协议，通过网络传输数据的。可以去http://archive.cloudera.com/cm5/repo-as-tarball/下载你所需要的cdh包。

- 解压安装包

下载完安装包后，解压到某个目录下，我这里下载的是cm5.4.10-ubuntu14-04.tar.gz这个版本。

```
tar -zxvf cm5.4.10-ubuntu14-04.tar.gz
chmod -R ugo+rX /opt/cm/local_resp/cm
```

解压后的目录是/opt/cm/local_resp/cm

- 启动Http server

启动一个Http服务，使可以通过网络来访问仓库中的数据。

```
cd /opt/cm/
python -m SimpleHTTPServer 8900
```

我这里使用的是8900，你可以根据需要，使用指定的端口。

- 验证

可在浏览器中访问地址http://server:8900/cm，如果可以正常访问，并且能看到相对应的文件列表，则表示正常启动。


#### 2.2.2.配置安装CM所需要的私有仓库

在目录**/etc/apt/sources.list.d/**下创建文件**my-private-cloudera-repo.list**，并写入配置把这个文件和上面新建的仓库关联到一起：

```
vi /etc/apt/sources.list.d/my-private-cloudera-repo.list

deb [arch=amd64] http://192.168.1.154:8900/cm/ trusty-cm5.4.10 contrib
```

执行下面的命令，使得上面的配置生效：

```
sudo apt-get update
```


### 2.2.3.配置使用CM安装节点时会用到的仓库
可以下载地址：http://archive.cloudera.com/cm5/ubuntu/trusty/amd64/cm/ 里的所有内容到本地，然后使用**2.2.1**中的方式来启动一个Http server。

### 2.2.4.配置节点的JDK
这一步是可选的，如果你不想每台机器都去手动安装，也可以在后边使用CM来批量安装。

### 2.3.开始安装CM
- 连接互联网安装

```
sudo ./cloudera-manager-installer.bin
```

或者

- 使用本地仓库安装

```
sudo ./cloudera-manager-installer.bin --skip_repo_package=1
```

然后就是一路的YES & NEXT，最后安装完成，CM默认的端口是7180，账户名和密码都是7180.


### 2.4.使用CM安装集群

首次登陆CM管理界面的时候，会出现一个集群安装向导。我这里选择的是免费版本。然后大概有如下几步：

- 使用ip或者hostname来搜索主机，搜索到之后，go to next step.

- 看到如下图片的时候，如果你已经在前面的主机中安装好了JDK，那么这里可以不选，如果没有，则必须选择。

![CM JDK](http://7xriy2.com1.z0.glb.clouddn.com/cm-02.png "cm jdk")

- 选择是否使用单用户模式（Single User Mode）

不使用该模式的话，HDFS服务会使用“hdfs”账户来启动，Hbase的Region Server会使用“hbase”账户来启动。使用了该模式之后，所有的服务都是使用同一个账户去启动的。
这里主要看集群的使用场景，如果其中涉及到不同的模块是由不同人员来运维管理的话，我建议还是不要使用单用户模式了。但如果集群是统一由一个人员来管理，那么选择使用单用户模式可能会方便很多。
我这里没有使用单用户模式。

- 配置好SSH登录

![配置ssh登录信息](http://7xriy2.com1.z0.glb.clouddn.com/cm-03.png "输入密码")

- 配置好SSH后开始连入主机，安装jdk和cm-agent
![install cm agent](http://7xriy2.com1.z0.glb.clouddn.com/cm-04.png "install cm agent")

- 安装完成，此时cm-agent就已经安装好了，随时都可以使用cm-server控制安装Hadoop的相关组件,完成后先不要点击进入下一步，继续看下面。
![cm agent ok](http://7xriy2.com1.z0.glb.clouddn.com/cm-05.png "cm agent ok")

- 分发Hadoop的安装包

去地址：

```
http://archive.cloudera.com/cdh5/parcels/5.4.10/
```

下载：

```
CDH-5.4.10-1.cdh5.4.10.p0.16-trusty.parcel
CDH-5.4.10-1.cdh5.4.10.p0.16-trusty.parcel.sha1
manifest.json
```

下载完成后：

```
把"CDH-5.4.10-1.cdh5.4.10.p0.16-trusty.parcel.sha1"重命名为"CDH-5.4.10-1.cdh5.4.10.p0.16-trusty.parcel.sha"
```

并移到目录：

```
/opt/cloudera/parcel-repo
```

然后在cm-serve点击进入下一页，将会看到如下图：
![cm parcel distributed](http://7xriy2.com1.z0.glb.clouddn.com/cm-06.png "cm parcel distributed")
因为你已经把安装包下载好，并且放入到/opt/cloudera/parcel-repo（默认的目录）里面，所以这里的**Download**自然就是100%，**Distributed**是把安装包从cm-server往集群中各个节点分发的过程，**Unpacked**是表示各个节点的上安装包的解压情况进度，前面都OK后，**Activated**自然就可以了，表示安装包已经部署好了，可以随时进行安装。

- 选择安装Hadoop的那些组件之后，会在这里显示各个组件部署的主机地址。
![hosts choose](http://7xriy2.com1.z0.glb.clouddn.com/cm-07.png "hosts choose")

- 这里配置的是一些组件使用的mysql信息：
![hadoop mysql config](http://7xriy2.com1.z0.glb.clouddn.com/cm-08.png "hadoop mysql config")

- 然后可以配置一些服务的的具体参数
![hadoop config](http://7xriy2.com1.z0.glb.clouddn.com/cm-09.png "hadoop config")

- 参数配置都没问题后，开始执行安装。
![hadoop install](http://7xriy2.com1.z0.glb.clouddn.com/cm-10.png "hadoop install")

- 安装完成后。
![install success](http://7xriy2.com1.z0.glb.clouddn.com/cm-11.jpeg "install success")
PS：我这里的Kafka是后来安装上去的，默认Hadoop的parcel包是没有kafka的。


## 3.注意事项

### 3.1.主机的Host配置不能出差错
我就配置错过host，期间走了一些弯路，主要表现在添加完主机之后，Hosts下的主机名都是localhost。

### 3.2.cm-agent安装失败重试时
如果在重试的过程中出现了一直等待，或者cm-agent端口被占用的情况，大概有下面的几种情况：

- 锁文件没有删除

```
sudo rm /tmp/.scm_prepare_node.lock
```

- 端口被占用 **9000/9001**

这种情况是部分进程没有关闭成功，找到端口对应的进程号，然后然后使用**kill -9停止进程**

```
netstat -nap | grep 9000
netstat -nap | grep 9001
```

主要是**supervisor**和**cm-agent**进程

## 4.会用到的一些地址总结

### 4.1.Mysql的相关配置
http://www.cloudera.com/documentation/enterprise/5-4-x/topics/cm_ig_mysql.html

### 4.2.创建本地仓库

- For CM

http://www.cloudera.com/documentation/enterprise/5-4-x/topics/cm_ig_create_local_package_repo.html

- For parcel

http://www.cloudera.com/documentation/enterprise/5-4-x/topics/cm_ig_create_local_parcel_repo.html

### 4.3.parcel的下载地址

http://archive.cloudera.com/cdh5/parcels/
http://archive.cloudera.com/kafka/parcels/

### 4.4.cm的下载地址

加压后可以直接运行
http://archive.cloudera.com/cm5/cm/5/

## 5.附件

博文中的图片是压缩后的，清晰度比较低，原图可以访问下面的地址：

链接: http://pan.baidu.com/s/1cmiBCE 密码: rnmw
