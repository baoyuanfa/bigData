

# Hadoop常用的调优参数

## 1．资源相关参数

### （1）以下参数是在用户自己的MR应用程序中配置就可以生效（mapred-default.xml）

| 配置参数                                     | 参数说明                                     |
| ---------------------------------------- | ---------------------------------------- |
| mapreduce.map.memory.mb                  | 一个MapTask可使用的资源上限（单位:MB），默认为1024。如果MapTask实际使用的资源量超过该值，则会被强制杀死。 |
| mapreduce.reduce.memory.mb               | 一个ReduceTask可使用的资源上限（单位:MB），默认为1024。如果ReduceTask实际使用的资源量超过该值，则会被强制杀死。 |
| mapreduce.map.cpu.vcores                 | 每个MapTask可使用的最多cpu core数目，默认值: 1         |
| mapreduce.reduce.cpu.vcores              | 每个ReduceTask可使用的最多cpu core数目，默认值: 1      |
| mapreduce.reduce.shuffle.parallelcopies  | 每个Reduce去Map中取数据的并行数。默认值是5               |
| mapreduce.reduce.shuffle.merge.percent   | Buffer中的数据达到多少比例开始写入磁盘。默认值0.66           |
| mapreduce.reduce.shuffle.input.buffer.percent | Buffer大小占Reduce可用内存的比例。默认值0.7            |
| mapreduce.reduce.input.buffer.percent    | 指定多少比例的内存用来存放Buffer中的数据，默认值是0.0          |

### （2）应该在YARN启动之前就配置在服务器的配置文件中才能生效（yarn-default.xml）

| 配置参数                                     | 参数说明                           |
| ---------------------------------------- | ------------------------------ |
| yarn.scheduler.minimum-allocation-mb     | 给应用程序Container分配的最小内存，默认值：1024 |
| yarn.scheduler.maximum-allocation-mb     | 给应用程序Container分配的最大内存，默认值：8192 |
| yarn.scheduler.minimum-allocation-vcores | 每个Container申请的最小CPU核数，默认值：1    |
| yarn.scheduler.maximum-allocation-vcores | 每个Container申请的最大CPU核数，默认值：32   |
| yarn.nodemanager.resource.memory-mb      | 给Containers分配的最大物理内存，默认值：8192  |

#### （3）Shuffle性能优化的关键参数，应在YARN启动之前就配置好（mapred-default.xml）

| 配置参数                             | 参数说明                   |
| -------------------------------- | ---------------------- |
| mapreduce.task.io.sort.mb        | Shuffle的环形缓冲区大小，默认100m |
| mapreduce.map.sort.spill.percent | 环形缓冲区溢出的阈值，默认80%       |

## 2．容错相关参数(MapReduce性能优化)

| 配置参数                         | 参数说明                                     |
| ---------------------------- | ---------------------------------------- |
| mapreduce.map.maxattempts    | 每个Map Task最大重试次数，一旦重试参数超过该值，则认为Map Task运行失败，默认值：4。 |
| mapreduce.reduce.maxattempts | 每个Reduce Task最大重试次数，一旦重试参数超过该值，则认为Map Task运行失败，默认值：4。 |
| mapreduce.task.timeout       | Task超时时间，经常需要设置的一个参数，该参数表达的意思为：如果一个Task在一定时间内没有任何进入，即不会读取新的数据，也没有输出数据，则认为该Task处于Block状态，可能是卡住了，也许永远会卡住，为了防止因为用户程序永远Block住不退出，则强制设置了一个该超时时间（单位毫秒），默认是600000。如果你的程序对每条输入数据的处理时间过长（比如会访问数据库，通过网络拉取数据等），建议将该参数调大，该参数过小常出现的错误提示是“AttemptID:attempt_14267829456721_123456_m_000224_0  Timed out after 300 secsContainer killed by the ApplicationMaster.”。 |