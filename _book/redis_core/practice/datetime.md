# 时间序列数据

与发生时间相关的一组数据，就是时间序列数据。这些数据的特点是**没有严格的关系模型，记录的信息可以表示成键和值的关系**

## 时间序列数据的读写特点

- 写：插入数据快
- 读：查询模式多
  - 单记录查询
  - 按时间范围查询
  - 时间范围的聚合计算

针对时间序列数据的“写要快”，Redis的高性能写特性直接就可以满足了；

针对“查询模式多”，也就是要支持单点查询、范围查询和聚合计算，Redis提供了保存时间序列数据的两种方案，分别可以基于Hash和Sorted Set实现，以及基于RedisTimeSeries模块实现。

## 基于Hash和Sorted Set保存时间序列数据

用Hash类型来实现单键的查询很简单。但是，Hash类型有个短板：它并不支持对数据进行范围查询。底层结构是哈希表，并没有对数据进行有序索引。

使用Sorted Set保存数据，可以把时间戳作为Sorted Set集合的元素分数，把时间点上记录的数据作为元素本身。就可以使用ZRANGEBYSCORE命令实现时间范围查询

同时使用Hash和Sorted Set，可以满足单个时间点和一个时间范围内的数据查询需求了。

### 如何保证写入Hash和Sorted Set是一个原子性的操作呢？

这里就涉及到了Redis用来实现简单的事务的MULTI和EXEC命令：

- MULTI命令：表示一系列原子性操作的开始。收到这个命令后，Redis就知道，接下来再收到的命令需要放到一个内部队列中，后续一起执行，保证原子性。
- EXEC命令：表示一系列原子性操作的结束。一旦Redis收到了这个命令，就表示所有要保证原子性的命令操作都已经发送完成了。此时，Redis开始执行刚才放到内部队列中的所有命令操作。

### 如何对时间序列数据进行聚合计算？

因为Sorted Set只支持范围查询，无法直接进行聚合计算，所以只能先把时间范围内的数据取回到客户端，然后在客户端自行完成聚合计算。这样的话数据在Redis实例和客户端间频繁传输，和其他操作命令竞争网络资源，导致其他操作变慢。

为了避免客户端和Redis实例间频繁的大量数据传输，我们可以使用RedisTimeSeries来保存时间序列数据。

## 基于RedisTimeSeries模块保存时间序列数据

RedisTimeSeries是redis的一个扩展模块，支持直接对数据进行按时间范围的聚合计算

因为RedisTimeSeries不属于Redis的内建功能模块，在使用时，我们需要先把它的源码单独编译成动态链接库redistimeseries.so，再使用loadmodule命令进行加载，如下所示：

```bash
loadmodule redistimeseries.so
```

使用示例如下：

```bash
# 创建一个key为device:temperature、数据有效期为600s的时间序列数据集合,设置了一个标签属性{device_id:1}
TS.CREATE device:temperature RETENTION 600000 LABELS device_id 1
OK

# 添加数据
TS.ADD device:temperature 1596416700 25.1
1596416700

# 读取最新数据
TS.GET device:temperature 
25.1

# 按标签过滤查询集合
TS.MGET FILTER device_id!=2 
1) 1) "device:temperature:1"
   2) (empty list or set)
   3) 1) (integer) 1596417000
      2) "25.3"
2) 1) "device:temperature:3"
   2) (empty list or set)
   3) 1) (integer) 1596417000
      2) "29.5"
3) 1) "device:temperature:4"
   2) (empty list or set)
   3) 1) (integer) 1596417000
      2) "30.1"

# 按照每180s的时间窗口，对时间段内的数据进行均值计算
TS.MGET FILTER device_id!=2 
1) 1) "device:temperature:1"
   2) (empty list or set)
   3) 1) (integer) 1596417000
      2) "25.3"
2) 1) "device:temperature:3"
   2) (empty list or set)
   3) 1) (integer) 1596417000
      2) "29.5"
3) 1) "device:temperature:4"
   2) (empty list or set)
   3) 1) (integer) 1596417000
      2) "30.1"
```
