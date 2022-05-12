# 消息队列

redis适合做消息队列吗？
这个问题的背后，隐含着两方面的核心问题：
- 消息队列的消息存取需求是什么？
- Redis如何实现消息队列的需求？

##  消息队列的消息存取需求

1. 消息保序
2. 重复消息处理
3. 消息可靠性保证

Redis的List和Streams两种数据类型，就可以满足消息队列的这三个需求。

## 基于List的消息队列解决方案

List本身就是按的先进先出顺序对数据进行存取的，所以**先天满足保序**需求。

具体来说，生产者可以使用LPUSH命令写入消息，消费者使用RPOP命令从List的另一端读取消息。

**潜在风险：** 生产者写入数据时，List并不主动通知消费者，如果消费者想及时处理消息，就得不停的（while 1循环）调用RPOP，带来不必要的性能损失。

**解决方案：** 使用BRPOP命令，阻塞式读取，客户端在没有读到队列数据时，自动阻塞，直到有新的数据写入队列，再开始读取新数据，节省了cpu开销。

**对于消息保序问题：** 生产者要生成全局唯一的ID，LPUSH时将ID带上，消费者读取数据时，根据ID来判断，避免消费重复数据

**可靠性：** 为了留存消息，List类型提供了BRPOPLPUSH命令，作用是让消费者程序从List读出消息，同事，Redis会把该消息再插入另一个List留存，这样一来，如果消费者读取了消息但没正常处理，等它重启后，就能从备份List中重新读取消息并处理了。

## 基于Streams的消息队列解决方案

Streams是Redis专门为消息队列设计的数据类型，它提供了丰富的消息队列操作命令。

- XADD：插入消息，保证有序，可自动生成全局唯一ID；
  
  ```bash
  # 往名称为mqstream的消息队列插入一条消息
  # 消息的key是repo，value是5；
  # 消息队列名称后面的 * 表示redis为插入的数据自动生成全局唯一ID；
  XADD mqstream * repo 5
  "1599203861727-0"
  # 如上所示：自动生成的ID由两部分组成：毫秒时间戳与消息序号（从0开始）
  ```
- XREAD：用于读取消息，可以按ID读取数据
  
  ```bash
  # XREAD在读取消息时，可以指定一个消息ID，并从这个消息ID的下一条消息开始进行读取。
  XREAD BLOCK 100 STREAMS mqstream 1599203861727-0
  1) 1) "mqstream"
   2) 1) 1) "1599274912765-0"
         2) 1) "repo"
            2) "3"
      2) 1) "1599274925823-0"
         2) 1) "repo"
            2) "2"
      3) 1) "1599274927910-0"
         2) 1) "repo"
            2) "1"
  ```
  另外，消费者也可以在调用XRAED时设定block配置项，实现类似于BRPOP的阻塞读取操作。当消息队列中没有消息时，一旦设置了block配置项，XREAD就会阻塞，阻塞的时长可以在block配置项进行设置。

  ```bash
  # 命令最后的$表示读取最新的消息
  # block 10000的配置项：无消息时阻塞10000ms再返回
  XREAD block 10000 streams mqstream $
  (nil)
  (10.00s)
  ```
- XREADGROUP：按消费组形式读取消息
  ```bash
  # 创建名为group1的消费组
  XGROUP create mqstream group1 0
  OK
  
  # 让group1消费组里的消费者consumer1从mqstream中读取所有消息
  # 其中，命令最后的参数“>”，表示从第一条尚未被消费的消息开始读取。
  XREADGROUP group group1 consumer1 streams mqstream >
  ```
  **消息队列中的消息一旦被消费组里的一个消费者读取了，就不能再被该消费组内的其他消费者读取了。**

- XPENDING：查询每个消费组内所有消费者已读取但未确认的消息
  ```bash
  # 第二、三行分别表示group2中所有消费者读取的消息最小ID和最大ID。
  XPENDING mqstream group2
  1) (integer) 3
  2) "1599203861727-0"
  3) "1599274925823-0"
  4) 1) 1) "consumer1"
        2) "1"
     2) 1) "consumer2"
        2) "1"
     3) 1) "consumer3"
        2) "1"
  # 如果还需要进一步查看某个消费者具体读取了哪些数据，可以执行下面的命令：
  XPENDING mqstream group2 - + 10 consumer2
  ```
- XACK：用于向消息队列确认消息处理已完成
  ```bash
  XACK mqstream group2 1599274912765-0
  ```

## 总结

!|基于List|基于Streams
:--:|:--:|:--:
消息保序|使用LPUSH/RPOP|使用XADD/XREAD
阻塞读取|使用BRPOP|使用XREAD block
重复消息处理|生产者自行实现全局唯一ID|Stream自动生成全局唯一ID
消息可靠性|使用BRPOPLPUSH|使用PENDING List自动留存消息，使用XPENDING查看，使用XACK确认消息
使用场景|Redis5.0前版本的部署环境，消息总量小|Redis5.0后的版本，消息总量大，需要消费组形式读取数据
