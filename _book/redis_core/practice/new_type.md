# 自定义数据类型

如何自定义数据类型？
一个自定义单链表类型示例如下描述：

**1. 定义新数据类型的底层结构**

   用newtype.h保存新类型的定义

   ```C
    struct NewTypeObject {
        struct NewTypeNode *head; // 头节点
        size_t len;     // 长度
    }NewTypeObject;
   ```
   其中，NewTypeNode结构就是我们自定义的新类型的底层结构
   ```C
   struct NewTypeNode {
        long value; // 保存实际数据
        struct NewTypeNode *next;   // 指向下一节点
    };
   ```

**2. 在RedisObject的type属性中，增加这个新类型的定义**
   
   这个定义是在Redis的server.h文件中。比如，增加一个叫作OBJ_NEWTYPE的宏定义，用来在代码中指代NewTypeObject这个新类型。

   ```C
   #define OBJ_STRING 0    /* String object. */
   #define OBJ_LIST 1      /* List object. */
   #define OBJ_SET 2       /* Set object. */
   #define OBJ_ZSET 3      /* Sorted set object. */
   …
   #define OBJ_NEWTYPE 7
   ```

**3. 开发新类型的创建和释放函数**
   
   redis把数据类型的创建和释放函数都定义在了object.c文件中，我们可以在该文件中增加NewTypeObject的创建函数createNewTypeObject，如下所示：

   ```C
   robj *createNewTypeObject(void){
       NewTypeObject *h = newtypeNew(); 
       robj *o = createObject(OBJ_NEWTYPE,h);
       return o;
   }
   ```

   newtypeNew函数涉及到新数据类型的具体创建。redis默认会为每个数据类型定义一个单独文件，实现这个类型的创建和命令操作，如t_string.c和t_list.c分别对应String和List类型。按照Redis的惯例，把newtypeNew函数定义在名为t_newtype.c的文件中。

   ```C
   NewTypeObject *newtypeNew(void){
       NewTypeObject *n = zmalloc(sizeof(*n));
       n->head = NULL;
       n->len = 0;
       return n;
   }
   ```

   createObject是Redis本身提供的RedisObject创建函数，它的参数是数据类型的type和指向数据类型实现的指针*ptr。

   ```C
   robj *createObject(int type, void *ptr) {
       robj *o = zmalloc(sizeof(*o));
       o->type = type;
       o->ptr = ptr;
       ...
       return o;
   }
   ```

   对于释放函数来说，它是创建函数的反过程，是用zfree命令把新结构的内存空间释放掉。

**4. 开发新类型的命令操作**

   1. 在t_newtype.c文件中增加命令操作的实现。比如说，我们定义ntinsertCommand函数，由它实现对NewTypeObject单向链表的插入操作：
   
        ```C
        void ntinsertCommand(client *c){
            //基于客户端传递的参数，实现在NewTypeObject链表头插入元素
        }
        ```
   2. 在server.h文件中，声明已经实现的命令，以便在server.c文件引用这个命令，例如：
        
        ```C
        void ntinsertCommand(client *c)
        ```
    
   3. 在server.c文件中的redisCommandTable里面，把新增命令和实现函数关联起来。例如，新增的ntinsert命令由ntinsertCommand函数实现，就可以用ntinsert命令给NewTypeObject数据类型插入元素了。

        ```C
        struct redisCommand redisCommandTable[] = { 
            ...
            {"ntinsert",ntinsertCommand,2,"m",...}
        }
        ```

此时，就完成了一个自定义的NewTypeObject数据类型，可以实现基本的命令操作了。如果还希望新的数据类型能被持久化保存，需要在Redis的RDB和AOF模块中增加对新数据类型进行持久化保存的代码
