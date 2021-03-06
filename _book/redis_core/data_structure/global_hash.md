# 全局hash表

上文说到的数据类型和数据结构都是value的底层实现，那么key和value怎么组织起来的呢？

为了从key到value的快速访问，redis采用了一个哈希表来保存所有键值对。称为**全局哈希表**，hash桶中的entry元素中保存了\*key和\*value指针，分别指向实际的键值（RedisObject）。

## 链式Hash

当需要根据key找到value时，会对key进行hash计算得到哈希桶位置，当哈希表数据逐渐变多时，难免会出现哈希冲突，redis采用链式Hash来解决哈希冲突，即：**一个哈希桶中的多个元素用一个链表来保存，它们之间依次用指针连接。**

## rehash

当哈希冲突越来越多，会导致哈希链越来越长，导致查找效率降低，就需要rehash：增加现有哈希桶的数量，以减少单个桶的冲突。

redis默认使用了2个全局哈希表：哈希表1和哈希表2，一开始默认使用哈希表1，此时哈希表2并没有分配空间，随着数据不断增多，redis开始rehash，过程如下：

1. 给哈希表2分配更大的空间，例如是当前哈希表1大小的两倍；
2. 把哈希表1中的数据重新映射并拷贝到哈希表2中；
3. 释放哈希表1的空间。

## 渐进式rehash

由于上述rehash过程的第2步涉及大量数据拷贝，如果一次性把哈希表1中的数据都迁移完，会造成redis主线程阻塞，redis采用了渐进式rehash来解决这个问题。

其实就是在第二步时，redis正常处理客户端请求，每处理一个请求，就从哈希表1中的第一个索引位置开始，将其上的entry链表拷贝到哈希表2上，处理下一个请求时，再拷贝下一个索引位置的entrys。
