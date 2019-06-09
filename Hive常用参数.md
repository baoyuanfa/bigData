# Hive常用参数

~~~
SET hive.optimize.skewjoin = true;
SET hive.skewjoin.key = 100000;
SET hive.exec.dynamic.partition.mode = nonstrict;
SET mapred.reducer.tasks = 50;
~~~

### Hive中间结果压缩和压缩输出

~~~
SET hive.exec.compress.output = true; -- 默认false
SET hive.exec.compress.intermediate = true; -- 默认false
SET mapred.output.compression.codec = org.apache.hadoop.io.compress.SnappyCodec; -- 默认org.apache.hadoop.io.compress.DefaultCodec
SET mapred.output.compression.type = BLOCK; -- 默认BLOCK
~~~

### 输出合并小文件

~~~
SET hive.merge.mapfiles = true; -- 默认true，在map-only任务结束时合并小文件
SET hive.merge.mapredfiles = true; -- 默认false，在map-reduce任务结束时合并小文件
SET hive.merge.size.per.task = 268435456; -- 默认256M
SET hive.merge.smallfiles.avgsize = 16777216; -- 当输出文件的平均大小小于该值时，启动一个独立的map-reduce任务进行文件merge
~~~

### 设置map和reduce数量

~~~
SET mapred.max.split.size = 256000000;
SET mapred.min.split.size = 64000000;
SET mapred.min.split.size.per.node = 64000000;
SET mapred.min.split.size.per.rack = 64000000;
SET hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
SET hive.exec.reducers.bytes.per.reducer = 256000000; -- 默认64M，每个reducer处理的数据量大小
~~~

### 设置数据倾斜和并行化

~~~
SET hive.exec.parallel = true; -- 并行执行
SET hive.exec.parallel.thread.number = 16;
SET mapred.job.reuse.jvm.num.tasks = 10;
SET hive.exec.dynamic.partition = true;
SET hive.optimize.cp = true; -- 列裁剪
SET hive.optimize.pruner = true; -- 分区裁剪
SET hive.groupby.skewindata = true; -- groupby数据倾斜
SET hive.exec.mode.local.auto = true; --本地执行
SET hive.exec.mode.local.auto.input.files.max = 10; //map数默认是4，当map数小于10就会启动任务本地执行
SET hive.exec.mode.local.auto.inputbytes.max = 128000000 --默认是128M
~~~

### 关闭以下两个参数来完成关闭Hive任务的推测执行

~~~
SET mapred.map.tasks.speculative.execution=false;
SET mapred.reduce.tasks.speculative.execution=false;
~~~

