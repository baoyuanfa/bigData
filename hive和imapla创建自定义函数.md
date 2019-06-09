# hive和imapla创建自定义函数

## 1.hive自定义UDF函数

​	1.导入依赖(版本和集群中一致)

~~~dependency
<dependencies>
		<!-- https://mvnrepository.com/artifact/org.apache.hive/hive-exec -->
		<dependency>
			<groupId>org.apache.hive</groupId>
			<artifactId>hive-exec</artifactId>
			<version>1.2.1</version>
		</dependency>
</dependencies>
~~~

​	2.自定义一个类继承UDF,手动实现其中的evaluate方法

~~~java
public class Lower extends UDF {

	public String evaluate (final String s) {
		
		if (s == null) {
			return null;
		}
		
		return s.toLowerCase();
	}
}

~~~

​	3.打包上传集群

​	4.在hive中添加函数

~~~hql
add jar jar包位置;
~~~

​	5.在hive中创建函数

~~~hql
create [temporary] function [dbname.]function_name AS class_name

-- class_name 为自定义函数全类名
~~~

## 2.自定义impalaUDF函数

​	1.2.3步骤与hive相同

​	4.将jar包上传到hdfs中

​	5.在impala中创建函数

~~~
create function mylower(string) returns string location 'jar包所在hdfs路径' symbol=自定义类的全类名;
~~~

