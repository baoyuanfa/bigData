# HBase

## 1.HBase概述

### 	1.什么是HBase

​		HBase的原型是Google的BigTable论文

​		HBase是Hadoop的生态系统，是建立在Hadoop文件系统（HDFS）之上的分布式、面向列的数据库，通过利用Hadoop的文件系统提供容错能力

### 	2.HBase特点

​		1.海量存储,  2.列式存储(列族)    3.极易扩展   4.高并发   5.稀疏(存储单位为cell,  cell中可以不存任何值, 如果存储都是一二进制格式存储)



## 2.HBase配置

### 	1.hbase-env.sh修改内容

~~~
export JAVA_HOME=/opt/module/jdk
export HBASE_MANAGES_ZK=false
~~~

### 	2.hbase-site.xml修改内容

~~~xml
<configuration>
	<property>     
		<name>hbase.rootdir</name>     
		<value>hdfs://hadoop102:9000/hbase</value>   
	</property>

	<property> 
		<name>hbase.cluster.distributed</name>
		<value>true</value>
	</property>

   <!-- 0.98后的新变动，之前版本没有.port,默认端口为60000 -->
	<property>
		<name>hbase.master.port</name>
		<value>16000</value>
	</property>

	<property>   
		<name>hbase.zookeeper.quorum</name>
	     <value>hadoop102:2181,hadoop103:2181,hadoop104:2181</value>
	</property>

	<property>   
		<name>hbase.zookeeper.property.dataDir</name>
	     <value>/opt/module/zookeeper-3.4.10/zkData</value>
	</property>
</configuration>
~~~

### 3.regionservers

~~~
Hadoop集群节点
~~~

### 4.软连接hadoop配置文件到hbase

​	ln -s

### Hbase页面端口号:16010

## 3.HBase  shell操作

~~~
查看当前数据库中有哪些表
list

创建表
create'student','info'

插入数据到表
put 'student','1001','info:sex','male'

扫描查看表数据
scan 'student'
scan 'student',{STARTROW => '1001', STOPROW  => '1001'}
scan 'student',{STARTROW => '1001'}

查看表结构
describe ‘student’

更新指定字段的数据
put 'student','1001','info:name','Nick'

查看“指定行”或“指定列族:列”的数据
get 'student','1001'
get 'student','1001','info:name'

统计表数据行数
count 'student'

删除某rowkey的全部数据
deleteall 'student','1001'
删除某rowkey的某一列数据
delete 'student','1002','info:sex'
清空表数据(清空表的操作顺序为先disable，然后再truncate)
truncate 'student'


删除表
disable 'student'
drop 'student'

变更表信息
将info列族中的数据存放3个版本
alter 'student',{NAME=>'info',VERSIONS=>3}
get 'student','1001',{COLUMN=>'info:name',VERSIONS=>3}
~~~

## 4.HBase API操作

### 	1.依赖

~~~
<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-server</artifactId>
    <version>1.3.1</version>
</dependency>

<dependency>
    <groupId>org.apache.hbase</groupId>
    <artifactId>hbase-client</artifactId>
    <version>1.3.1</version>
</dependency>

<dependency>
	<groupId>jdk.tools</groupId>
	<artifactId>jdk.tools</artifactId>
	<version>1.8</version>
	<scope>system</scope>
	<systemPath>${JAVA_HOME}/lib/tools.jar</systemPath>
</dependency>
~~~

### 	2.HBaseAPI

​	表操作:

~~~java
Configuration conf = HBaseConfiguration.create();
conf.set("hbase.zookeeper.quorum", "192.168.9.102");
conf.set("hbase.zookeeper.property.clientPort", "2181");

//判断表是否存在
HBaseAdmin admin = new HBaseAdmin(conf);
admin.tableExists(tableName);

//创建表
HTableDescriptor descriptor = new HTableDescriptor(TableName.valueOf(tableName));
descriptor.addFamily(new HColumnDescriptor(cf));
admin.createTable(descriptor);

//删除表
admin.disableTable(tableName);
admin.deleteTable(tableName);
~~~

​	数据操作:

~~~java
//向表中插入数据
//创建HTable对象
HTable hTable = new HTable(conf, tableName);
//向表中插入数据
Put put = new Put(Bytes.toBytes(rowKey));
//向Put对象中组装数据
put.add(Bytes.toBytes(columnFamily), Bytes.toBytes(column), Bytes.toBytes(value));
hTable.put(put);
hTable.close();


//删除多行数据
List<Delete> deleteList = new ArrayList<Delete>();
for(String row : rows){
	Delete delete = new Delete(Bytes.toBytes(row));
	deleteList.add(delete);
}
hTable.delete(deleteList);
hTable.close();

//获取所有数据
Scan scan = new Scan();
//使用HTable得到resultcanner实现类的对象
ResultScanner resultScanner = hTable.getScanner(scan);
for(Result result : resultScanner){
	Cell[] cells = result.rawCells();
	for(Cell cell : cells){
		//得到rowkey
		System.out.println("行键:" + Bytes.toString(CellUtil.cloneRow(cell)));
		//得到列族
		Bytes.toString(CellUtil.cloneFamily(cell));
		Bytes.toString(CellUtil.cloneQualifier(cell));
		Bytes.toString(CellUtil.cloneValue(cell));
	}
}

//获取某一行数据
Get get = new Get(Bytes.toBytes(rowKey));
Result result = table.get(get);
for(Cell cell : result.rawCells()){
		Bytes.toString(result.getRow()));
		Bytes.toString(CellUtil.cloneFamily(cell));
		Bytes.toString(CellUtil.cloneQualifier(cell));
		Bytes.toString(CellUtil.cloneValue(cell));
		cell.getTimestamp();
}

//获取某一行指定“列族:列”的数据
Get get = new Get(Bytes.toBytes(rowKey));
get.addColumn(Bytes.toBytes(family), Bytes.toBytes(qualifier));
Result result = table.get(get);
for(Cell cell : result.rawCells()){
		Bytes.toString(result.getRow));
		Bytes.toString(CellUtil.cloneFamily(cell));
		Bytes.toString(CellUtil.cloneQualifier(cell));
		Bytes.toString(CellUtil.cloneValue(cell));
}

~~~



### 3.MapReduce

#### 	1.环境变量的导入

​	在/etc/profile配置

~~~
export HBASE_HOME=/opt/module/hbase
export HADOOP_HOME=/opt/module/hadoop-2.7.2
~~~

​	hadoop-env.sh (在for循环之后配)

~~~
export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:/opt/module/hbase/lib/*
~~~

-- 案例: 使用MapReduce将hdfs数据导入到HBase

​		1.创建HBase表            create 'fruit','info'

​		2.HDFS中创建input_fruit文件夹并上传fruit.tsv文件

​		3.执行MapReduce到HBase的fruit表中

~~~shell
/opt/module/hadoop-2.7.2/bin/yarn jar lib/hbase-server-1.3.1.jar importtsv \
-Dimporttsv.columns=HBASE_ROW_KEY,info:name,info:color fruit \
hdfs://hadoop102:9000/input_fruit
~~~

### 2.自定义MapReduce

~~~
mapper方法
extends TableMapper<ImmutableBytesWritable, Put>

reducer方法
TableReducer<ImmutableBytesWritable, Put, NullWritable>
~~~

## 5.与Hive集成

### 1.环境准备

~~~shell
export HBASE_HOME=/opt/module/hbase
export HIVE_HOME=/opt/module/hive

ln -s $HBASE_HOME/lib/hbase-common-1.3.1.jar  $HIVE_HOME/lib/hbase-common-1.3.1.jar
ln -s $HBASE_HOME/lib/hbase-server-1.3.1.jar $HIVE_HOME/lib/hbase-server-1.3.1.jar
ln -s $HBASE_HOME/lib/hbase-client-1.3.1.jar $HIVE_HOME/lib/hbase-client-1.3.1.jar
ln -s $HBASE_HOME/lib/hbase-protocol-1.3.1.jar $HIVE_HOME/lib/hbase-protocol-1.3.1.jar
ln -s $HBASE_HOME/lib/hbase-it-1.3.1.jar $HIVE_HOME/lib/hbase-it-1.3.1.jar
ln -s $HBASE_HOME/lib/htrace-core-3.1.0-incubating.jar $HIVE_HOME/lib/htrace-core-3.1.0-incubating.jar
ln -s $HBASE_HOME/lib/hbase-hadoop2-compat-1.3.1.jar $HIVE_HOME/lib/hbase-hadoop2-compat-1.3.1.jar
ln -s $HBASE_HOME/lib/hbase-hadoop-compat-1.3.1.jar $HIVE_HOME/lib/hbase-hadoop-compat-1.3.1.jar
~~~

​	hive-site.xml中修改zookeeper的属性

~~~xml
<property>
  <name>hive.zookeeper.quorum</name>
  <value>hadoop102,hadoop103,hadoop104</value>
  <description>The list of ZooKeeper servers to talk to. This is only needed for read/write locks.</description>
</property>
<property>
  <name>hive.zookeeper.client.port</name>
  <value>2181</value>
  <description>The port of ZooKeeper servers to talk to. This is only needed for read/write locks.</description>
</property>
~~~

### 2.案例

#### 1.案例一

​	 建立Hive表，关联HBase表，插入数据到Hive表的同时能够影响HBase表

 1). 在Hive中创建表同时关联HBase

~~~sql
CREATE TABLE hive_hbase_emp_table(
empno int,
ename string,
job string,
mgr int,
hiredate string,
sal double,
comm double,
deptno int)
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES ("hbase.columns.mapping" = ":key,info:ename,info:job,info:mgr,info:hiredate,info:sal,info:comm,info:deptno")
TBLPROPERTIES ("hbase.table.name" = "hbase_emp_table");
~~~

2). 在Hive中创建临时中间表，用于load文件中的数据

~~~sql
CREATE TABLE emp(
empno int,
ename string,
job string,
mgr int,
hiredate string,
sal double,
comm double,
deptno int)
row format delimited fields terminated by '\t';


load data local inpath '/opt/module/data/emp.txt' into table emp;


insert into table hive_hbase_emp_table select * from emp;


select * from hive_hbase_emp_table;


scan ‘hbase_emp_table’
~~~

#### 2．案例二 

​	在HBase中已经存储了某一张表hbase_emp_table，然后在Hive中创建一个外部表来关联HBase中的hbase_emp_table这张表，使之可以借助Hive来分析HBase这张表中的数据。

 1).在Hive中创建外部表

~~~sql
CREATE EXTERNAL TABLE relevance_hbase_emp(
empno int,
ename string,
job string,
mgr int,
hiredate string,
sal double,
comm double,
deptno int)
STORED BY 
'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES ("hbase.columns.mapping" = 
":key,info:ename,info:job,info:mgr,info:hiredate,info:sal,info:comm,info:deptno") 
TBLPROPERTIES ("hbase.table.name" = "hbase_emp_table");
~~~