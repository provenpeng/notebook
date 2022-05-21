# AOF日志

## AOF日志的内容

如下所示，*3表示当前命令有3个部分，每部分由“$+数字”开头，后面紧跟命令，这里“数字“表示该部分中命令 or 键 or 值有多少字节。当redis重启需要恢复数据时，就会读取AOF文件中的命令，逐一执行。

>内容示例：*3$3set$7testkey$9testvalue

## 写后日志

与mysql的redo log的WAL（写前日志）不同，AOF机制是先执行命令，将数据写入内存，再记录日志。

此方式有几个优点：

1. 保证AOF日志中的命令是正确的（命令报错直接返回，避免记录错误的命令）
2. 命令执行后才记录日志，不会阻塞当前操作

潜在风险：

1. 当redis刚执行完一个命令，还来不及记录日志就宕机了，那该命令和相应的数据有丢失风险
2. AOF虽然避免了对当前命令的阻塞，但可能会阻塞下个命令。因为AOF日志（落盘）也是在主线程执行的

写后日志的风险均来自于AOF写入磁盘的时机，redis提供了如下3种写回策略，可通过*appendfsync*配置项设置：
·
配置项|写回时机|优点|缺点
:--:|:--:|:--:|:--:
Always|同步写回|可靠性高，数据基本不丢失|每个写命令都要落盘，性能影响较大
Everysec|每秒写回|性能适中|宕机时丢失1秒数据
No|操作系统控制写回|性能好|宕机时丢失数据较多

## 日志文件太大了怎么办？——AOF重写机制

AOF是以追加的方式，逐一记录收到的命令，当一个键值对被多条写命令反复修改时，AOF文件会记录相应的多条命令，导致文件越来越大。

AOF重写机制就是，redis根据数据库现状创建一个新的AOF文件，读取库中所有键值对，对每一个键值对用一条命令记录它的写入。起到了”多变一“的效果。

## AOF重写会阻塞主线程吗？

不会，因为重写过程是后台线程bgrewriteaof完成的。

重写可以概括为 ”一个拷贝，两处日志“

一个拷贝是指：重写时主线程fork出后台的bgrewriteaof子进程，此时，fork会把主线程的内存拷贝一份给**bgrewriteaof**，其中包含了数据库中的最新数据，就能进行AOF重写了

两处日志是指：

1. 正在使用的AOF日志，redis会将新的写操作日志写入其缓冲区，防止重写期间宕机丢日志
2. 新的AOF重写日志，新的写操作也会被写到重写日志缓冲区中，重写完成后，将重写缓冲区的最新操作写入新的AOF文件，此时，新的AOF文件就可以将旧的AOF文件替换掉了

## AOF重写的潜在风险

主要风险有2点：

1. fork子进程的瞬间，会阻塞主进程；fork时不会一次性copy所有内存数据给子进程，而是使用OS提供的**写时复制**（Copy On Write）机制，避免一次性copy大量数据而阻塞。
2. 写时拷贝发生时，拷贝bigkey可能会阻塞。