# Redis作为缓存的问题

由于Redis用作缓存的普遍性以及它在业务应用中的重要作用，所以需要系统地掌握缓存的一系列内容，包括工作原理、替换策略、异常处理和扩展机制。

## 一、Redis作为缓存是怎么工作的？

### 1. 缓存的特征

计算机系统中，默认有两种缓存：
- CPU里面的末级缓存，即LLC，用来缓存内存中的数据，避免每次从内存中存取数据；
- 内存中的高速页缓存，即page cache，用来缓存磁盘中的数据，避免每次从磁盘中存取数据。

跟内存相比，LLC的访问速度更快，而跟磁盘相比，内存的访问是更快的。

可以看出缓存具有如下特征：

1. 在一个层次化的系统中，缓存一定是一个快速子系统，数据存在缓存中时，能避免每次从慢速子系统中存取数据。
2. 缓存系统的容量大小总是小于后端慢速系统的，不可能把所有数据都放在缓存系统中。

### 2. Redis缓存处理请求的两种情况

业务应用在访问数据时，会先查询Redis中是否保存了相应的数据：
1. 缓存命中：直接读取，性能贼快
2. 缓存缺失：需要从后端数据库中读数据，性能会变慢。而且需要将缺失数据写入redis（缓存更新），这会涉及到缓存一致性问题。

### 3. 旁路缓存

如果应用程序想要使用Redis缓存，就要在程序中增加相应的缓存操作代码。所以，我们也把Redis称为旁路缓存。因为需要新增程序代码来使用缓存，所以，Redis并不适用于那些无法获得源码的应用。

### 4. 缓存类型

按照Redis缓存是否接受写请求，我们可以把它分成只读缓存和读写缓存。

1. 只读缓存
   
   读取数据时，会先调Redis GET接口，查询数据是否存在。而所有的数据写请求，会直接法网后端数据库，在数据库中进行增删改。对于删改的数据来说，如果redis已经缓存了相应的数据，应用需要把这些缓存的数据删除。

   **优点：** 所有最新的数据都在数据库中，不会有丢失的风险。当我们需要缓存图片、短视频这些用户只读数据时，就可以使用只读缓存这个类型了。

2. 读写缓存
   
   读请求的处理过程和只读缓存一样，而写请求也会被发送到缓存，在缓存中直接进行增删改操作。

   **优点：** 提升业务应用的响应速度。
   **缺点：** 最新的数据在redis中，有丢失风险。

   因此，根据业务应用对数据可靠性和缓存性能的不同要求，有同步直写（保证数据可靠性）、异步写回（提供快速响应）两种策略，以下进行详细说明：

   - 同步直写：写请求发给缓存的同时，也会发给后端数据库进行处理，等到缓存和数据库都写完数据，才给客户端返回
   - 异步写回：所有写请求先在缓存中处理，等到这些增改的数据要被从缓存中淘汰出来时，缓存将他们写回后端数据库。

## 二、缓存不一致问题


## 三、缓存雪崩、击穿、穿透


## 四、缓存污染


## 五、Pika-基于SSD实现大容量Redis