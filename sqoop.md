# sqoop

## 1.RDBMS到HDFS

​	全部导入:

~~~
bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password root \
--table staff \
--target-dir /user/company \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by "\t"
~~~

​	查询导入:

~~~
bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password root \
--target-dir /user/company \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by "\t" \
--query 'select name,sex from staff where id <=1 and $CONDITIONS;'
~~~

​	导入指定列:

~~~
bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--target-dir /user/company \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by "\t" \
--columns id,sex \
--table staff
~~~

​	使用sqoop关键字筛选查询导入数据:

~~~
bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--target-dir /user/company \
--delete-target-dir \
--num-mappers 1 \
--fields-terminated-by "\t" \
--table staff \
--where "id=1"
~~~

## 2.RDBMS到Hive

~~~
bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table staff \
--num-mappers 1 \
--hive-import \
--fields-terminated-by "\t" \
--hive-overwrite \
--hive-table staff_hive
~~~

## 3.RDBMS到Hbase

~~~
bin/sqoop import \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table company \
--columns "id,name,sex" \
--column-family "info" \
--hbase-create-table \
--hbase-row-key "id" \
--hbase-table "hbase_company" \
--num-mappers 1 \
--split-by id
~~~

​	提示：sqoop1.4.6只支持HBase1.0.1之前的版本的自动创建HBase表的功能

​	解决方案：手动创建HBase表

## 4.导出数据

​	HIVE/HDFS到RDBMS:

~~~
bin/sqoop export \
--connect jdbc:mysql://hadoop102:3306/company \
--username root \
--password 000000 \
--table staff \
--num-mappers 1 \
--export-dir /user/hive/warehouse/staff_hive \
--input-fields-terminated-by "\t"
~~~

## 5.脚本打包

### 1) 创建一个.opt文件

~~~
vim ob_HDFS2RDBMS.opt
~~~

### 2)编写sqoop脚本

~~~
export
--connect jdbc:mysql://hadoop102:3306/company
--username root
--password root
--table staff
--num-mappers 1
--export-dir /user/hive/warehouse/staff_hive
--input-fields-terminated-by "\t"
~~~

### 3) 执行该脚本

~~~
 bin/sqoop --options-file opt/job_HDFS2RDBMS.opt
~~~



