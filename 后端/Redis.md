## 数据类型

#### 通用

https://redis.io/commands/?group=generic

Expire,del,exists,keys,ttl

#### String

https://redis.io/commands/?group=string

set,get,mset,mget,incr,incrby

#### Hash

HSET key field value

HGET key field

HMSET

HMGET

HGETALL key

HKEYS  key 获取某个key的所有field

HVALUES

#### List



#### Set







## 非阻塞IO

读写方法不会阻塞， 而是能读多少读多少， 能写多少写多少







## RDB



## AOF

所有的**修改性**指令序列

**先执行指令才将曰志存盘**

AOF瘦身：遍历内存，转换成一系列Redis操作指令