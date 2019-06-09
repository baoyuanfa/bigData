# Hadoop 集群搭建

## 一、虚拟机准备

### 1.创建三台虚拟机（集群规模3台为例）

1）、配置HOSTNAME（vim /etc/sysconfig/network）

​	将HOSTNAME修改为自定义的主机名

2）、配置eth0

1. vim /etc/udev/rules.d/70-persistent-net.rules
   1. 删除eth0所在行、并将eth1为eth0
2. vim /etc/sysconfig/network-scripts/ifcfg-eth0
   1. DEVICE=eth0
      TYPE=Ethernet
      ONBOOT=yes
      BOOTPROTO=static
      NAME="eth0"
      IPADDR=192.168.a.b	(a与设置的网段相同，b为任意值0~255)
      PREFIX=24
      GATEWAY=192.168.a.2
      DNS1=192.168.a.2

可以定义脚本reset

~~~shell
#!/bin/bash
cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=static
NAME="eth0"
IPADDR=192.168.1.$1
PREFIX=24
GATEWAY=192.168.1.2
DNS1=192.168.1.2
EOF

sed -i '/eth0/d' /etc/udev/rules.d/70-persistent-net.rules
sed -i 's/eth1/eth0/g' /etc/udev/rules.d/70-persistent-net.rules

sed -i "s/HOSTNAME=.*/HOSTNAME=hadoop$1/" /etc/sysconfig/network

reboot
~~~



### 2.修改 /etc/hosts文件

1、添加hostname 以及对应的ip

例如：192.168.1.100   hadoop100

```txt
192.168.1.101   hadoop101
192.168.1.102   hadoop102
192.168.1.103   hadoop103
192.168.1.104   hadoop104
192.168.1.105   hadoop105
192.168.1.106   hadoop106
192.168.1.107   hadoop107
192.168.1.108   hadoop108
192.168.1.109   hadoop109
```
### 3.创建用户并设置密码

~~~shell
useradd bigdata
passwd bigdata
~~~

### 4.配置这个用户为sudoers

​	在/etc/sudoers中root下方添加bigdata权限

~~~shell
vim /etc/sudoers

root    ALL=(ALL)       ALL
bigdata    ALL=(ALL)       NOPASSWD:ALL
~~~

### 5.创建软件安装目录并修改拥有者

~~~shell
mkdir /opt/module /opt/software
chown bigdata:bigdata /opt/module /opt/software
~~~



## 二、框架基本选型

| 框架            | 版本       |
| ------------- | -------- |
| Hadoop        | 2.7.2    |
| Flume         | 1.7.0    |
| Kafka         | 0.11.0.2 |
| Kafka Manager | 1.3.3.22 |
| Hive          | 1.2.1    |
| Sqoop         | 1.4.6    |
| MySQL         | 5.6.24   |
| Azkaban       | 2.5.0    |
| Java          | 1.8      |
| Zookeeper     | 3.4.10   |
| Presto        | 0.189    |

## 三、安装jdk、hadoop

### 1.上传、解压

​	配置环境变量 /etc/profile 

​	source /etc/profile

​	测试是否成功 	java -version，hadoop version

​	成功后分发至hadoop03、hadoop04(可在ssh配置之后执行)

~~~shell
scp /etc/profile root@hadoop103:/etc/profile

scp /etc/profile root@hadoop104:/etc/profile

~~~



### 2.修改hadoop配置（hadoop目录下的etc/hadoop)

1. hadoop-env.sh/mapred-env.sh/yarn-env.sh

   1. 修改其中的JAVA_HOME

2. core-site.xml

   ~~~~xml
   <!-- 指定HDFS中NameNode的地址 -->
   <property>
   		<name>fs.defaultFS</name>
         <value>hdfs://hadoop102:9000</value>
   </property>

   <!-- 指定Hadoop运行时产生文件的存储目录 -->
   <property>
   		<name>hadoop.tmp.dir</name>
   		<value>/opt/module/hadoop-2.7.2/data/tmp</value>
   </property>
   ~~~~

   ​

3. hdfs-site.xml

   ~~~xml
   <!-- 副本数 -->
   <property>
   		<name>dfs.replication</name>
   		<value>3</value>
   </property>

   <!-- 指定Hadoop辅助名称节点主机配置 -->
   <property>
         <name>dfs.namenode.secondary.http-address</name>
         <value>hadoop104:50090</value>
   </property>
   ~~~

   ​

4. yarn-site.xml

   ~~~xml
   <!-- Reducer获取数据的方式 -->
   <property>
   		<name>yarn.nodemanager.aux-services</name>
   		<value>mapreduce_shuffle</value>
   </property>

   <!-- 指定YARN的ResourceManager的地址 -->
   <property>
   		<name>yarn.resourcemanager.hostname</name>
   		<value>hadoop103</value>
   </property>
   ~~~




   <!-- 日志聚集功能使能 -->
           <property>
               <name>yarn.log-aggregation-enable</name>
               <value>true</value>
           </property>
    
           <!-- 日志保留时间设置7天 -->
           <property>
               <name>
                 yarn.log-aggregation.retain-seconds
             	</name>
               <value>604800</value>
           </property>
   ~~~

   ​

5. mapred-site.xml

   需要将mapred-site.xml.template改名为mapred-site.xml

   ~~~shell
   cp mapred-site.xml.template mapred-site.xml

   vim mapred-site.xml
   ~~~

   ~~~xml
   <!-- 指定MR运行在Yarn上 -->
   <property>
   		<name>mapreduce.framework.name</name>
   		<value>yarn</value>
   </property>


   <!-- 历史服务器端地址 -->
       <property>
           <name>mapreduce.jobhistory.address</name>
           <value>hadoop104:10020</value>
       </property>
       <!-- 历史服务器web端地址 -->
       <property>
        	<name>
         		 mapreduce.jobhistory.webapp.address
         	</name>
           <value>hadoop104:19888</value>
       </property>
   ~~~

6. slaves

   ~~~xml
   hadoop102
   hadoop103
   hadoop104
   ~~~

### 3.配置ssh免密登录

~~~shell
#配置hadoop102、hadoop03
ssh-keygen -t rsa
#提示信息三次回车

ssh-copy-id hadoop102
ssh-copy-id hadoop103
ssh-copy-id hadoop104

~~~

### 4.编写分发脚本	（放入 /home/bigdata下）

~~~shell
#!/bin/bash

#判断是否传入参数
pcount=$#
if (($pcount==0))
then
	echo "no args"
	exit
fi

#获取输入的文件名
p1=$1
fname=`basename $p1`
echo fname=$fname

#获取文件父目录
pdir=`cd -P $(dirname $p1);pwd`
echo pdir=$pdir

#获取当前用户名
host=`whoami`

for i in hadoop102 hadoop103 hadoop104
do
	echo ========== $i ==========
	rsync -rvl $pdir/$fname $host@$i:$pdir/
done
~~~

### 5.分发jdk以及hadoop

~~~shell
xsync /opt/module/jdk
xsync /opt/module/hadoop
~~~



### 6.启动hadoop

​	1、格式化namenode(只能格式化一次，每次格式化都会生成一个集群cid)

~~~
hdfs namenode -format
~~~



​	2、群起集群

​	hadoop102上启动hdfs

~~~
start-dfs.sh
~~~



​	3、hadoop103上启动yarn

~~~
start-yarn.sh
~~~



​	4、hadoop104上启动historyserver

~~~
sbin/mr-(tab补全) start historyserver
~~~





### 7.配置集群时间同步

1.、配置时间服务器（必须root用户、安装ntpd服务）

​	1、修改/etc/ntp.conf

~~~
vim /etc/ntp.conf
~~~



​	修改内容如下：

~~~shell
#restrict 192.168.1.0mask 255.255.255.0 nomodify notrap改为
restrict 192.168.1.0 mask255.255.255.0 nomodify notrap


server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst改为
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst

#添加信息
server 127.127.1.0
fudge 127.127.1.0 stratum 10
~~~



​	2、修改/etc/sysconfig/ntpd

~~~shell
vim /etc/sysconfig/ntpd
~~~



~~~shell
#添加一行
SYNC_HWCLOCK=yes
~~~



​	3、重启ntpd服务并设置开机启动

~~~shell
service ntpd restart
chkconfig ntpd on
~~~

​	4、其他机器配置定时同步任务（root用户）

~~~shell
crontab -e
~~~



~~~shell
#添加定时任务
*/10 * * * * /usr/sbin/ntpdate hadoop102
~~~

### 8.安装zookeeper

​	1.上传、解压

​	2.修改配置文件

~~~shell
cp zoo_sample.cfg zoo.cfg

vim zoo.cfg
~~~

~~~shell
#修改datadir
datadir=/opt/module/zookeeper-3.4.10/zkData


#添加服务集群服务端口
server.2=hadoop102:2888:3888
server.3=hadoop103:2888:3888
server.4=hadoop104:2888:3888
~~~

​	配置解读

​	server.A=B:C:D。

​	A是一个数字，表示这个是第几号服务器；集群模式下配置一个文件myid，这	个文件在dataDir目录下，这个文件里面有一个数据就是A的值，Zookeeper启动时读取此文件，拿到里面的数据与zoo.cfg里面的配置信息比较从而判断到底是哪个server。

​	B是这个服务器的ip地址；

​	C是这个服务器与集群中的Leader服务器交换信息的端口；

​	D是万一集群中的Leader服务器挂了，需要一个端口来重新进行选举，选出一个新的Leader，而这个端口就是用来执行选举时服务器相互通信的端口

​	3.在zookeeper目录下创建zkData文件夹

~~~shell
mkdir zkaDta
~~~

​	4.在zkData目录下创建myid文件

~~~shell
vim myid
~~~

​	5.在myid中添加zookeeper节点id，不同节点myid不能相同

​	6.修改日志存放目录

~~~shell
vim bin/zkEnv.sh
~~~

~~~shell
#修改ZOO_LOG_DIR="."
ZOO_LOG_DIR="/opt/module/zookeeper/logs"
~~~



### 9、编写通用脚本

1.使用前需给当前用户添加使用环境变量权限

~~~shell
cat /etc/profile >> /home/bigdata/.bashrc
~~~



2.编写集群通用脚本(放入 /home/atguigu/bin 目录下)

​	1.编写xcall.sh

~~~
vim xcall.sh
~~~



~~~shell
#!/bin/bash

for i in hadoop102 hadoop103 hadoop104
do
	echo ========== $i ==========
	ssh $i "$*"
done
~~~



​	2.编写zookeeper群起、关闭脚本

~~~shell
vim zk.sh
~~~

~~~shell
#!/bin/bash

case $1 in
"start")
	for i in hadoop102 hadoop103 hadoop104
	do
		echo ========== $i ==========
		ssh $i "/opt/modulezookeeper-3.4.10/bin/zkServer.sh start"
	done
;;
"stop")
for i in hadoop102 hadoop103 hadoop104
	do
		echo ========== $i ==========
		ssh $i "/opt/modulezookeeper-3.4.10/bin/zkServer.sh stop"
	done
;;
*)
	echo "please input {start | stop}"
;;
esac
~~~

### 10、安装flume

1.上传解压

2.修改配置文件

​	将flume/conf下的flume-env.sh.template文件修改为flume-env.sh，并配置flume-env.sh文件

~~~shell
mv flume-env.sh.template flume-env.sh
~~~

~~~shell
vi flume-env.sh
export JAVA_HOME=/opt/module/jdk1.8.0_144
~~~

flume启动停止脚本

~~~shell
vim f1.sh
~~~

~~~shell
#! /bin/bash

case $1 in
"start"){
        for i in hadoop102 hadoop103
        do
                echo " --------启动 $i 采集flume-------"
                ssh $i "nohup /opt/module/flume/bin/flume-ng agent --conf-file /opt/module/flume/conf/file-flume-kafka.conf --name a1 -Dflume.root.logger=INFO,LOGFILE >/dev/null 2>&1 &"
        done
};;	
"stop"){
        for i in hadoop102 hadoop103
        do
                echo " --------停止 $i 采集flume-------"
                ssh $i "ps -ef | grep file-flume-kafka | grep -v grep |awk '{print \$2}' | xargs kill"
        done

};;
esac
~~~

说明：nohup，该命令可以在你退出帐户/关闭终端之后继续运行相应的进程。nohup就是不挂起的意思，不挂断地运行命令。

/dev/null代表linux的空设备文件，所有往这个文件里面写入的内容都会丢失，俗称“黑洞”。

标准输入0    从键盘获得输入 /proc/self/fd/0 

标准输出1    输出到屏幕（即控制台） /proc/self/fd/1 

错误输出2    输出到屏幕（即控制台） /proc/self/fd/2

3.安装Ganglia监控器

3.1 Ganglia的安装与部署

1)安装httpd服务与php

~~~shell
sudo yum -y install httpd php
~~~

2)安装其他依赖

~~~shell
sudo yum -y install rrdtool perl-rrdtool rrdtool-devel
sudo yum -y install apr-devel
~~~

3)**安装ganglia**

~~~shell
sudo rpm -Uvh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm

sudo yum -y install ganglia-gmetad
sudo yum -y install ganglia-web
sudo yum install -y ganglia-gmond
~~~

4)修改配置文件/etc/httpd/conf.d/ganglia.conf

~~~shell
sudo vim /etc/httpd/conf.d/ganglia.conf
~~~

修改以下标注的配置的配置：

~~~shell
# Ganglia monitoring system php web frontend
Alias /ganglia /usr/share/ganglia
<Location /ganglia>
  Order deny,allow
  Deny from all
  #========添加一行========
  Allow from all
  #========注释以下三行========
  # Allow from 127.0.0.1
  # Allow from ::1
  # Allow from .example.com
</Location>

~~~

5)修改配置文件/etc/ganglia/gmetad.conf

~~~shell
sudo vim /etc/ganglia/gmetad.conf
~~~

修改：

~~~shell
data_source "hadoop102" 192.168.1.102
~~~

6)修改配置文件/etc/ganglia/gmond.conf

~~~shell
sudo vim /etc/ganglia/gmond.conf
~~~

修改为：

~~~shell
cluster {
  #======修改name======
  name = "hadoop102"
  owner = "unspecified"
  latlong = "unspecified"
  url = "unspecified"
}
udp_send_channel {
  #bind_hostname = yes # Highly recommended, soon to be default.
                       # This option tells gmond to use a source address
                       # that resolves to the machine's hostname.  Without
                       # this, the metrics may appear to come from any
                       # interface and the DNS names associated with
                       # those IPs will be used to create the RRDs.
  #=====注释=====
  # mcast_join = 239.2.11.71
  #=====添加=====
  host = 192.168.1.102
  port = 8649
  ttl = 1
}
udp_recv_channel {
  #注释
  # mcast_join = 239.2.11.71
  port = 8649
  #======修改=====
  bind = 192.168.1.102
  retry_bind = true
  # Size of the UDP buffer. If you are handling lots of metrics you really
  # should bump it up to e.g. 10MB or even higher.
  # buffer = 10485760
}

~~~

7)修改配置文件/etc/selinux/config

~~~shell
sudo vim /etc/selinux/config
~~~

修改为：

~~~shell
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
#=======修改=======
SELINUX=disabled
# SELINUXTYPE= can take one of these two values:
#     targeted - Targeted processes are protected,
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
~~~

尖叫提示：selinux本次生效关闭必须重启，如果此时不想重启，可以临时生效之：

~~~shell
sudo setenforce 0
~~~

8)启动ganglia

~~~shell
sudo service httpd start
sudo service gmetad start
sudo service gmond start
~~~

9)打开网页浏览ganglia页面

http://192.168.1.102/ganglia

尖叫提示：如果完成以上操作依然出现权限不足错误，请修改/var/lib/ganglia目录的权限:

~~~shell
sudo chmod -R 777 /var/lib/ganglia
~~~

3.2操作Flume测试监控

1)修改/opt/module/flume/conf目录下的flume-env.sh配置:

~~~xml
JAVA_OPTS="-Dflume.monitoring.type=ganglia
-Dflume.monitoring.hosts=192.168.1.102:8649
-Xms100m
-Xmx200m"
~~~

2)启动Flume任务

~~~shell
bin/flume-ng agent \
--conf conf/ \
--name a1 \
--conf-file job/flume-telnet-logger.conf \
-Dflume.root.logger==INFO,console \
-Dflume.monitoring.type=ganglia \
-Dflume.monitoring.hosts=192.168.1.102:8649
~~~

### 11、kafka安装

1、集群规划

hadoop102                                  hadoop103                           hadoop104

zk                                              zk                                       zk

kafka                                         kafka                                  kafka

2、上传解压

3、在/opt/module/kafka目录下创建logs文件夹

~~~shell
mkdir logs
~~~

4、修改配置文件

~~~shell
vim conf/server.properties
~~~

输入以下内容：

~~~shell
#broker的全局唯一编号，不能重复
broker.id=0
#删除topic功能使能
delete.topic.enable=true
#处理网络请求的线程数量
num.network.threads=3
#用来处理磁盘IO的现成数量
num.io.threads=8
#发送套接字的缓冲区大小
socket.send.buffer.bytes=102400
#接收套接字的缓冲区大小
socket.receive.buffer.bytes=102400
#请求套接字的缓冲区大小
socket.request.max.bytes=104857600
#kafka运行日志存放的路径	
log.dirs=/opt/module/kafka/logs
#topic在当前broker上的分区个数
num.partitions=1
#用来恢复和清理data下数据的线程数量
num.recovery.threads.per.data.dir=1
#segment文件保留的最长时间，超时将被删除
log.retention.hours=168
#配置连接Zookeeper集群地址
zookeeper.connect=hadoop102:2181,hadoop103:2181,hadoop104:2181

~~~

5、配置环境变量

~~~shell
sudo vi /etc/profile
~~~

~~~xml
#KAFKA_HOME
export KAFKA_HOME=/opt/module/kafka
export PATH=$PATH:$KAFKA_HOME/bin
~~~

~~~shell
source /etc/profile
~~~

6、分发安装包

~~~shell
xsync kafka/
~~~

7、修改broker.id

分别在hadoop103和hadoop104上修改配置文件

/opt/module/kafka/config/server.properties中的broker.id=1、broker.id=2

注：broker.id不得重复

kafka启动停止脚本

~~~shell
vim kf.sh
~~~

~~~shell
#! /bin/bash

case $1 in
"start"){
        for i in hadoop102 hadoop103 hadoop104
        do
                echo " --------启动 $i kafka-------"
                # 用于KafkaManager监控

                ssh $i "export JMX_PORT=9988 && /opt/module/kafka/bin/kafka-server-start.sh -daemon /opt/module/kafka/config/server.properties "
        done
};;
"stop"){
        for i in hadoop102 hadoop103 hadoop104
        do
                echo " --------停止 $i kafka-------"
                ssh $i "/opt/module/kafka/bin/kafka-server-stop.sh stop"
        done
};;
esac

~~~



注意：启动Kafka时要先开启JMX端口，是用于后续KafkaManager监控。



8.kafka Manager安装

8.1、上传解压

8.2、进入到/opt/module/kafka-manager-1.3.3.22/conf目录，在application.conf文件中修改kafka-manager.zkhosts

~~~shell
vim application.conf
~~~

修改为

~~~shell
kafka-manager.zkhosts="hadoop102:2181,hadoop103:2181,hadoop104:2181"
~~~

8.3启动KafkaManager

~~~shell
nohup bin/kafka-manager   -Dhttp.port=7456 >/opt/module/kafka-manager-1.3.3.22/start.log 2>&1 &
~~~

8.4在浏览器中打开

http://hadoop102:7456

kafka Manager启动停止脚本

~~~shell
vim km.sh
~~~

~~~shell
#! /bin/bash

case $1 in
"start"){
        echo " -------- 启动 KafkaManager -------"
        nohup /opt/module/kafka-manager-1.3.3.22/bin/kafka-manager   -Dhttp.port=7456 >start.log 2>&1 &
};;
"stop"){
        echo " -------- 停止 KafkaManager -------"
        ps -ef | grep ProdServerStart | grep -v grep |awk '{print $2}' | xargs kill
};;
esac
~~~

8.5Kafka压力测试

8.5.1Kafka压测

​	用Kafka官方自带的脚本，对Kafka进行压测。Kafka压测时，可以查看到哪个地方出现了瓶颈（CPU，内存，网络IO）。一般都是网络IO达到瓶颈。 

​	kafka-consumer-perf-test.sh

​	kafka-producer-perf-test.sh

8.5.2Kafka Producer压力测试

​	在/opt/module/kafka/bin目录下面有这两个文件。我们来测试一下

~~~shell
bin/kafka-producer-perf-test.sh  --topic test --record-size 100 --num-records 100000 --throughput 1000 --producer-props bootstrap.servers=hadoop102:9092,hadoop103:9092,hadoop104:9092
~~~

​	说明：record-size是一条信息有多大，单位是字节。num-records是总共发送多少条信息。throughput 是每秒多少条信息。

8.5.3Kafka Consumer压力测试

​	Consumer的测试，如果这四个指标（IO，CPU，内存，网络）都不能改变，考虑增加分区数来提升性能。

~~~shell
bin/kafka-consumer-perf-test.sh --zookeeper hadoop102:2181 --topic test --fetch-size 10000 --messages 10000000 --threads 1
~~~

参数说明：

--zookeeper 指定zookeeper的链接信息

--topic 指定topic的名称

--fetch-size 指定每次fetch的数据的大小

--messages 总共要消费的消息个数

### 12、采集通道启动/停止脚本

1.在/home/byf/bin目录下创建脚本cluster.sh

~~~shell
vim cluster.sh
~~~

在脚本中填写如下内容

~~~shell
#! /bin/bash

case $1 in
"start"){
	echo " -------- 启动 集群 -------"

	echo " -------- 启动 hadoop集群 -------"
	/opt/module/hadoop-2.7.2/sbin/start-dfs.sh 
	ssh hadoop103 "/opt/module/hadoop-2.7.2/sbin/start-yarn.sh"

	#启动 Zookeeper集群
	zk.sh start

	#启动 Flume采集集群
	f1.sh start

	#启动 Kafka采集集群
	kf.sh start

sleep 4s;

	#启动 Flume消费集群
	f2.sh start

	#启动 KafkaManager
	km.sh start
};;
"stop"){
        echo " -------- 停止 集群 -------"

	#停止 KafkaManager
	km.sh stop

    #停止 Flume消费集群
	f2.sh stop

	#停止 Kafka采集集群
	kf.sh stop

    sleep 4s;

	#停止 Flume采集集群
	f1.sh stop

	#停止 Zookeeper集群
	zk.sh stop

	echo " -------- 停止 hadoop集群 -------"
	ssh hadoop103 "/opt/module/hadoop-2.7.2/sbin/stop-yarn.sh"
	/opt/module/hadoop-2.7.2/sbin/stop-dfs.sh 
};;
esac
~~~

### 13、hive安装

1.上传解压

2.修改/opt/module/hive/conf目录下的hive-env.sh.template名称为hive-env.sh

~~~shell
mv hive-env.sh.template hive-env.sh
~~~

3.配置hive-env.sh文件

~~~shell
#配置HADOOP_HOME路径
export HADOOP_HOME=/opt/module/hadoop-2.7.2
#配置HIVE_CONF_DIR路径
export HIVE_CONF_DIR=/opt/module/hive/conf
~~~

4.Hadoop集群配置

4.1.必须启动hdfs和yarn

~~~shell
sbin/start-dfs.sh
sbin/start-yarn.sh
~~~

4.2.在HDFS上创建/tmp和/user/hive/warehouse两个目录并修改他们的同组权限可写

~~~shel
bin/hadoop fs -mkdir /tmp
bin/hadoop fs -mkdir -p /user/hive/warehouse
bin/hadoop fs -chmod g+w /tmp
bin/hadoop fs -chmod g+w /user/hive/warehouse
~~~

5.MySql安装

5.1.上传解压

5.2.查看mysql是否安装，如果安装了，卸载mysql

~~~shell
rpm -qa|grep mysql
rpm -e --nodeps mysql.....
~~~

5.3.安装mysql

~~~shell
#安装服务端
rpm -ivh MySQL-server-5.6.24-1.el6.x86_64.rpm
#记住初始密码
cat /root/.mysql_secret
service mysql status
service mysql start

#安装客户端
rpm -ivh MySQL-client-5.6.24-1.el6.x86_64.rpm
mysql -uroot -p初始密码
~~~

5.4.修改mysql密码

~~~sql
SET PASSWORD=PASSWORD('000000');
~~~

5.5.修改user表，把Host表内容修改为%

~~~sql
update user set host='%' where host='localhost';
~~~

5.6.删除root用户的其他host

~~~sql
delete from user where Host='hadoop102';
delete from user where Host='127.0.0.1';
delete from user where Host='::1';
~~~

6.Hive元数据配置到MySql

6.1.驱动拷贝

​	拷贝mysql-connector-java-5.1.27-bin.jar到/opt/module/hive/lib/

6.2.配置Metastore到MySql

6.2.1 在/opt/module/hive/conf目录下创建一个hive-site.xml

~~~shell
vim hive-site.xml
~~~

​	根据官方文档配置参数，拷贝数据到hive-site.xml文件中

https://cwiki.apache.org/confluence/display/Hive/AdminManual+MetastoreAdmin

~~~xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
	<property>
	  <name>javax.jdo.option.ConnectionURL</name>
	  <value>jdbc:mysql://hadoop102:3306/metastore?createDatabaseIfNotExist=true</value>
	  <description>JDBC connect string for a JDBC metastore</description>
	</property>

	<property>
	  <name>javax.jdo.option.ConnectionDriverName</name>
	  <value>com.mysql.jdbc.Driver</value>
	  <description>Driver class name for a JDBC metastore</description>
	</property>

	<property>
	  <name>javax.jdo.option.ConnectionUserName</name>
	  <value>root</value>
	  <description>username to use against metastore database</description>
	</property>

	<property>
	  <name>javax.jdo.option.ConnectionPassword</name>
	  <value>root123</value>
	  <description>password to use against metastore database</description>
	</property>
</configuration>
~~~

7.Hive常见属性配置

7.1 Hive数据仓库位置配置

​	修改default数据仓库原始位置（将hive-default.xml.template如下配置信息拷贝到hive-site.xml文件中）

~~~xml
<property>
	<name>hive.metastore.warehouse.dir</name>
	<value>/user/hive/warehouse</value>
	<description>location of default database for the warehouse</description>
</property>
~~~

配置同组用户有执行权限

~~~shell
bin/hdfs dfs -chmod g+w /user/hive/warehouse
~~~

7.2 查询后信息显示配置

​	在hive-site.xml文件中添加如下配置信息，就可以实现显示当前数据库，以及查询表的头信息配置

~~~xml
<property>
	<name>hive.cli.print.header</name>
	<value>true</value>
</property>

<property>
	<name>hive.cli.print.current.db</name>
	<value>true</value>
</property>
~~~



7.3 Hive运行日志信息配置

7.3.1 Hive的log默认存放在/tmp/byf/hive.log目录下（当前用户名下）

7.3.2 修改hive的log存放日志到/opt/module/hive/logs

​	修改/opt/module/hive/conf/hive-log4j.properties.template文件名称为

hive-log4j.properties

~~~shell
 mv hive-log4j.properties.template hive-log4j.properties
~~~

在hive-log4j.properties文件中修改log存放位置

~~~xml
hive.log.dir=/opt/module/hive/logs
~~~



7.4 修改hive支持中文注释

7.4.1 修改hive-site.xml中的参数

~~~xml
<property>
	<name>javax.jdo.option.ConnectionURL</name>					<value>jdbc:mysql://hadoop102:3306/metastore?createDatabaseIfNotExist=true&amp;useUnicode=true&amp;characterEncoding=UTF-8
	</value>
	<description>JDBC connect string for a JDBC metastore</description>
</property>
~~~



7.4.2 进入数据库 Metastore 中执行以下 SQL 语句

1.修改表字段注解和表注解

~~~sql
alter table COLUMNS_V2 modify column COMMENT varchar(256) character set utf8;
alter table TABLE_PARAMS modify column PARAM_VALUE varchar(4000) character set utf8;
~~~

2.修改分区字段注解

~~~sql
alter table PARTITION_PARAMS modify column PARAM_VALUE varchar(4000) character set
utf8;
alter table PARTITION_KEYS modify column PKEY_COMMENT varchar(4000) character set utf8;
~~~

7.5  关闭元数据检查

~~~shell
vim hive-site.xml
~~~

~~~xml
<property>
    <name>hive.metastore.schema.verification</name>
    <value>false</value>
</property>
~~~

8.Hive运行引擎Tez

8.1 上传解压

8.2 在Hive中配置Tez

8.2.1 在/opt/module/hive/conf下修改hive-env.sh文件

添加如下配置：

~~~shell
# Set HADOOP_HOME to point to a specific hadoop install directory
export HADOOP_HOME=/opt/module/hadoop-2.7.2

# Hive Configuration Directory can be controlled by:
export HIVE_CONF_DIR=/opt/module/hive/conf

# Folder containing extra libraries required for hive compilation/execution can be controlled by:
export TEZ_HOME=/opt/module/tez-0.9.1    #是你的tez的解压目录
export TEZ_JARS=""
for jar in `ls $TEZ_HOME |grep jar`; do
    export TEZ_JARS=$TEZ_JARS:$TEZ_HOME/$jar
done
for jar in `ls $TEZ_HOME/lib`; do
    export TEZ_JARS=$TEZ_JARS:$TEZ_HOME/lib/$jar
done

export HIVE_AUX_JARS_PATH=/opt/module/hadoop-2.7.2/share/hadoop/common/hadoop-lzo-0.4.20.jar$TEZ_JARS
~~~

8.2.2 在hive-site.xml文件中添加如下配置，更改hive计算引擎

~~~xml
<property>
	<name>hive.execution.engine</name>
	<value>tez</value>
</property>
~~~

8.2.3 配置Tez

​	在Hive 的/opt/module/hive/conf下面创建一个tez-site.xml文件

~~~shell
vim tez-site.xml
~~~

​	添加如下内容：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<property>
	<name>tez.lib.uris</name>
  	<value>${fs.defaultFS}/tez/tez-0.9.1,${fs.defaultFS}/tez/tez-0.9.1/lib</value>
</property>
<property>
	<name>tez.lib.uris.classpath</name>    						<value>${fs.defaultFS}/tez/tez-0.9.1,${fs.defaultFS}/tez/tez-0.9.1/lib</value>
</property>
<property>
     <name>tez.use.cluster.hadoop-libs</name>
     <value>true</value>
</property>
<property>
	<name>tez.history.logging.service.class</name>	<value>org.apache.tez.dag.history.logging.ats.ATSHistoryLoggingService</value>
</property>
</configuration>
~~~

8.2.4 上传Tez到集群

​	将/opt/module/tez-0.9.1上传到HDFS的/tez路径

~~~shell
hadoop fs -mkdir /tez
~~~



~~~shell
hadoop fs -put /opt/module/tez-0.9.1/ /tez
~~~

8.2.5 运行Tez时检查到用过多内存而被NodeManager杀死进程问题

方案一：或者是关掉虚拟内存检查。我们选这个，修改yarn-site.xml

~~~xml
<property>
	<name>yarn.nodemanager.vmem-check-enabled</name>
	<value>false</value>
</property>
~~~

方案二：mapred-site.xml中设置Map和Reduce任务的内存配置如下：(value中实际配置的内存需要根据自己机器内存大小及应用情况进行修改)

~~~xml
<property>
　　<name>mapreduce.map.memory.mb</name>
　　<value>1536</value>
</property>
<property>
　　<name>mapreduce.map.java.opts</name>
　　<value>-Xmx1024M</value>
</property>
<property>
　　<name>mapreduce.reduce.memory.mb</name>
　　<value>3072</value>
</property>
<property>
　　<name>mapreduce.reduce.java.opts</name>
　　<value>-Xmx2560M</value>
</property>
~~~

### 14、sqoop安装

1、上传解压

2、修改配置文件

2.1 重命名配置文件

~~~shell
mv sqoop-env-template.sh sqoop-env.sh
~~~

2.2  修改配置文件(sqoop-env.sh)

~~~shell
export HADOOP_COMMON_HOME=/opt/module/hadoop-2.7.2
export HADOOP_MAPRED_HOME=/opt/module/hadoop-2.7.2
export HIVE_HOME=/opt/module/hive
export ZOOKEEPER_HOME=/opt/module/zookeeper-3.4.10
export ZOOCFGDIR=/opt/module/zookeeper-3.4.10
export HBASE_HOME=/opt/module/hbase
~~~

2.3 拷贝JDBC驱动到sqoop的lib目录下

~~~shell
cp mysql-connector-java-5.1.27-bin.jar /opt/module/sqoop-1.4.6.bin__hadoop-2.0.4-alpha/lib/
~~~

2.4 测试Sqoop是否能够成功连接数据库

~~~shell
bin/sqoop list-databases --connect jdbc:mysql://hadoop102:3306/ --username root --password 000000
~~~

3、常用导入命令

~~~sqoop
/opt/module/sqoop/bin/sqoop import \
--connect  \
--username  \
--password  \
--target-dir  \
--delete-target-dir \
--num-mappers   \
--fields-terminated-by   \
--query   "$2"' and  $CONDITIONS;'
~~~

### 15、azkaban

1、上传解压

2、安装Azkaban

2.1、在/opt/module/目录下创建azkaban目录

~~~shell
mkdir azkaban
~~~

2.2、对解压后的文件重新命名

~~~shell
mv azkaban-web-2.5.0/ server
mv azkaban-executor-2.5.0/ executor
~~~

2.3、azkaban脚本导入

​	进入mysql，创建azkaban数据库，并将解压的脚本导入到azkaban数据库

~~~sql
[byf@hadoop102 azkaban]$ mysql -uroot -p000000
mysql> create database azkaban;
mysql> use azkaban;
mysql> source /opt/module/azkaban/azkaban-2.5.0/create-all-sql-2.5.0.sql
~~~

2.4、生成密钥库

​	Keytool是java数据证书的管理工具，使用户能够管理自己的公/私钥对及相关证书。

​	-keystore    指定密钥库的名称及位置(产生的各类信息将不在.keystore文件中)

​	-genkey      在用户主目录中创建一个默认文件".keystore" 

​	-alias  对我们生成的.keystore 进行指认别名；如果没有默认是mykey

​	-keyalg  指定密钥的算法 RSA/DSA 默认是DSA

2.4.1、生成 keystore的密码及相应信息的密钥库

~~~shell
keytool -keystore keystore -alias jetty -genkey -keyalg RSA
~~~

​	注意：密钥库的密码至少必须6个字符，可以是纯数字或者字母或者数字和字母的组合等等密钥库的密码最好和<jetty> 的密钥相同，方便记忆

2.5、时间同步配置（一般不需要配置）

​	2.5.1、如果在[/usr/share/zoneinfo/]()这个目录下不存在时区配置文件Asia/Shanghai，就要用 tzselect 生成

~~~shell
tzselect
~~~

​	选择Asia/China/Beijing Time

​	2.5.2、拷贝该时区文件，覆盖系统本地时区配置

~~~shell
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
~~~

3、配置文件

3.1、Web服务器配置

3.1.1、进入azkaban web服务器安装目录 conf目录，打开azkaban.properties文件

~~~shell
vim azkaban.properties
~~~

3.1.2、按照如下配置修改azkaban.properties文件

~~~properties
#Azkaban Personalization Settings
#服务器UI名称,用于服务器上方显示的名字
azkaban.name=Test
#描述
azkaban.label=My Local Azkaban
#UI颜色
azkaban.color=#FF3601
azkaban.default.servlet.path=/index
#默认web server存放web文件的目录
web.resource.dir=/opt/module/azkaban/server/web/
#默认时区,已改为亚洲/上海 默认为美国
default.timezone.id=Asia/Shanghai

#Azkaban UserManager class
user.manager.class=azkaban.user.XmlUserManager
#用户权限管理默认类（绝对路径）
user.manager.xml.file=/opt/module/azkaban/server/conf/azkaban-users.xml

#Loader for projects
#global配置文件所在位置（绝对路径）
executor.global.properties=/opt/module/azkaban/executor/conf/global.properties
azkaban.project.dir=projects

#数据库类型
database.type=mysql
#端口号
mysql.port=3306
#数据库连接IP
mysql.host=hadoop102
#数据库实例名
mysql.database=azkaban
#数据库用户名
mysql.user=root
#数据库密码
mysql.password=000000
#最大连接数
mysql.numconnections=100

# Velocity dev mode
velocity.dev.mode=false

# Azkaban Jetty server properties.
# Jetty服务器属性.
#最大线程数
jetty.maxThreads=25
#Jetty SSL端口
jetty.ssl.port=8443
#Jetty端口
jetty.port=8081
#SSL文件名（绝对路径）
jetty.keystore=/opt/module/azkaban/server/keystore
#SSL文件密码
jetty.password=000000
#Jetty主密码与keystore文件相同
jetty.keypassword=000000
#SSL文件名（绝对路径）
jetty.truststore=/opt/module/azkaban/server/keystore
#SSL文件密码
jetty.trustpassword=root123

# Azkaban Executor settings
executor.port=12321

# mail settings
mail.sender=
mail.host=
job.failure.email=
job.success.email=

lockdown.create.projects=false

cache.directory=cache

~~~

3.1.3、web服务器用户配置

​	在azkaban web服务器安装目录 conf目录，按照如下配置修改[azkaban-users.xml]() 文件，增加管理员用户

~~~xml
[byf@hadoop102 conf]$ vim azkaban-users.xml
<azkaban-users>
	<user username="azkaban" password="azkaban" roles="admin" groups="azkaban" />
	<user username="metrics" password="metrics" roles="metrics"/>
	<user username="admin" password="admin" roles="admin,metrics" />
	<role name="admin" permissions="ADMIN" />
	<role name="metrics" permissions="METRICS"/>
</azkaban-users>
~~~

3.2、执行服务器配置

3.2.1、进入执行服务器安装目录conf，打开azkaban.properties

~~~properties
#Azkaban
#时区
default.timezone.id=Asia/Shanghai

# Azkaban JobTypes Plugins
#jobtype 插件所在位置
azkaban.jobtype.plugin.dir=plugins/jobtypes

#Loader for projects
executor.global.properties=/opt/module/azkaban/executor/conf/global.properties
azkaban.project.dir=projects

database.type=mysql
mysql.port=3306
mysql.host=hadoop102
mysql.database=azkaban
mysql.user=root
mysql.password=root123
mysql.numconnections=100

# Azkaban Executor settings
#最大线程数
executor.maxThreads=50
#端口号(如修改,请与web服务中一致)
executor.port=12321
#线程数
executor.flow.threads=30
~~~

3.2.2、启动executor服务器

​	在executor服务器目录下执行启动命令

~~~shell
bin/azkaban-executor-start.sh
~~~

3.3、启动web服务器

​	在azkaban web服务器目录下执行启动命令

~~~shell
bin/azkaban-web-start.sh
~~~

https://hadoop102:8443





