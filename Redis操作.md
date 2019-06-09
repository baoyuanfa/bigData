# Redis操作

## 1.key

~~~
1.keys  *
	查询当前库的所有键
2.exists  <key>
	判断某个键是否存在
3.type  <key>   
	查看键的类型
4.del  <key>
	删除某个键
5.expire   <key>   <seconds>
	为键值设置过期时间，单位秒。
6.ttl   <key> 
	查看还有多少秒过期，-1表示永不过期，-2表示已过期
7.dbsize 
	查看当前数据库的key的数量
8.flushdb
	清空当前库
9.flushall
	通杀全部库 
~~~

## 2.String

~~~
1.get   <key>
	查询对应键值
2.set   <key>  <value>
	添加键值对
3.append  <key>  <value>
	将给定的<value> 追加到原值的末尾
4.strlen  <key>
	获得值的长度
5.setnx  <key>  <value>
	只有在 key 不存在时设置 key 的值
6.incr  <key>
	将 key 中储存的数字值增1
	只能对数字值操作，如果为空，新增值为1
7.decr  <key>
	将 key 中储存的数字值减1
	只能对数字值操作，如果为空，新增值为-1
8.incrby / decrby  <key>  <步长>
	将 key 中储存的数字值增减。自定义步长
9.mset  <key1>  <value1>  <key2>  <value2>  ..... 
	同时设置一个或多个 key-value对  
10.mget  <key1>   <key2>   <key3> ..... 
	同时获取一个或多个 value  
11.msetnx <key1>  <value1>  <key2>  <value2>  ..... 
	同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在
12.getrange  <key>  <起始位置>  <结束位置>
	获得值的范围，类似java中的substring
13.setrange  <key>   <起始位置>   <value>
	用 <value>  覆写<key> 所储存的字符串值，从<起始位置>开始
14.setex  <key>  <过期时间>   <value>
	设置键值的同时，设置过期时间，单位秒
15.getset <key>  <value>
	以新换旧，设置了新值同时获得旧值

~~~

## 3.List

~~~
1.lpush/rpush  <key>  <value1>  <value2>  <value3> ....
	从左边/右边插入一个或多个值
2.lpop/rpop  <key> 
	从左边/右边吐出一个值(等同于删除该值)
3.rpoplpush  <key1>  <key2>  
	从<key1>列表右边吐出一个值，插到<key2>列表左边。
4.lrange <key> <start> <stop>
	按照索引下标获得某一组中的元素(从左到右)
5.lindex <key> <index>
	按照索引下标获得元素(从左到右)
6.llen <key>
	获得列表长度 
7.linsert <key>  before <value>  <newvalue>
	在<value>的前面插入<newvalue>
8.lrem <key> <n>  <value>
	从左边删除n个value(从左到右)
~~~

## 4.Set

~~~
1.sadd <key>  <value1>  <value2> .....   
	将一个或多个 member 元素加入到集合 key 当中，已经存在于集合的 member 元素将被忽略
2.smembers <key>
	取出该集合的所有值
3.sismember <key>  <value>
	判断集合<key>是否为含有该<value>值，有返回1，没有返回0
4.scard   <key>
	返回该集合的元素个数
5.srem <key> <value1> <value2> ....
	删除集合中的某个元素
6.spop <key>  
	随机从该集合中吐出一个值
7.srandmember <key> <n>
	随机从该集合中取出n个值。,不会从集合中删除
8.sinter <key1> <key2>  
	返回两个集合的交集元素
9.sunion <key1> <key2>  
	返回两个集合的并集元素
10.sdiff <key1> <key2>  
	返回两个集合的差集元素
~~~

## 5.Hash

~~~
1.hset <key>  <field>  <value>
	给<key>集合中的  <field>键赋值<value>
2.hget <key1>  <field>   
	从<key1>集合<field> 取出 value 
3.hmset <key1>  <field1> <value1> <field2> <value2>... 
	批量设置hash的值
4.hexists key  <field>
	查看哈希表 key 中，给定域 field 是否存在
5.hkeys <key>   
	列出该hash集合的所有field
6.hvals <key>    
	列出该hash集合的所有value
7.hincrby <key> <field>  <increment> 
	为哈希表 key 中的域 field 的值加上增量 increment 
8.hsetnx <key>  <field> <value>
	将哈希表 key 中的域 field 的值设置为 value ，当且仅当域 field 不存在
~~~

## 6.ZSet

~~~
1.zadd  <key> <score1> <value1>  <score2> <value2>...
	将一个或多个 member 元素及其 score 值加入到有序集 key 当中。
2.zrange <key>  <start> <stop>  [WITHSCORES]   
	返回有序集 key 中，下标在<start> <stop>之间的元素
	带WITHSCORES，可以让分数一起和值返回到结果集
3.zrangebyscore key min max [withscores] [limit offset count]
	返回有序集 key 中，所有 score 值介于 min 和 max 之间(包括等于 min 或 max )的成员。有序集成员按 score 值递增(从小到大)次序排列
4.zrevrangebyscore key max min [withscores] [limit offset count]
	同上，改为从大到小排列。 
5.zincrby <key> <increment> <value>
	为元素的score加上增量
6.zrem  <key>  <value>  
	删除该集合下，指定值的元素 
7.zcount <key>  <min>  <max> 
	统计该集合，分数区间内的元素个数 
8.zrank <key>  <value> 
	返回该值在集合中的排名，从0开始
~~~

