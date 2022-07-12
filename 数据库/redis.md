# redis

基于内存的kv数据库，由c实现

## 数据结构

redis使用自身的类型系统：

* redisObject对象
* 基于 redisObject 对象的类型检查.
* 基于 redisObject 对象的显式多态函数.
* 对 redisObject 进行分配、共享和销毁的机制

一个redisobject包括以下这些属性：

```c
/*
 * Redis 对象
 */
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 编码方式
    unsigned encoding:4;

    // LRU - 24位, 记录最末一次访问时间（相对于lru_clock）; 或者 LFU（最少使用的数据：8位频率，16位访问时间）
    unsigned lru:LRU_BITS; // LRU_BITS: 24

    // 引用计数
    int refcount;

    // 指向底层数据结构实例
    void *ptr;
} robj;
/*
* 对象类型
*/
#define OBJ_STRING 0 // 字符串
#define OBJ_LIST 1 // 列表
#define OBJ_SET 2 // 集合
#define OBJ_ZSET 3 // 有序集
#define OBJ_HASH 4 // 哈希表

/*
* 对象编码
*/
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* 注意：版本2.6后不再使用. */
#define OBJ_ENCODING_LINKEDLIST 4 /* 注意：不再使用了，旧版本2.x中String的底层之一. */
#define OBJ_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of ziplists */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
```

redisOject对象池：共享字符串对象，如10000以内的数字

### string

存储的值可以是字符串、整数或者浮点数

底层实现为char数组的封装，称为SDS。

结构基本为 |len|alloc|flags|buf|

len是字符串的长度，alloc是分配的内存长度，flags标记头部类型，因为头部类型有uint8、u16，u32，u64几种类型。

扩容：小于1m则两倍扩容，大于1m则每次扩容1m，最大512m。

存储：

小于44个字符时，标志位为EMBSTR，sds与redisobject连续存储，大小为64B
大于44个字符时，标志位位RAW，使用ptr指针指向sds，当可以解析为数字时，标志位INT，存储与ptr指针内。

有点类似stl的string库。这种封装可以常数复杂度获取字符串长度，减少缓冲区溢出。

SDS有空间与分配和惰性空间释放两种策略。

* 空间与分配：对字符串进行空间扩展的时候，扩展的内存比实际需要的多，这样可以减少连续执行字符串增长操作所需的内存重分配次数。
* 惰性空间释放：对字符串进行缩短操作时，程序不立即使用内存重新分配来回收缩短后多余的字节，而是使用 alloc 属性将这些字节的数量记录下来，等待后续使用。

可以用来做缓存、session共享

### list

底层：quicklist

redis是内存数据库，相对于其他基于磁盘的，内存的空间比较紧缺。因此设计了压缩性质的ziplist

3.0之后list键已经不直接用ziplist和linkedlist作为底层实现了，取而代之的是quicklist

ziplist是为了提高存储效率而设计的一种特殊编码的双向链表。它可以存储字符串或者整数，存储整数时是采用整数的二进制而不是字符串形式存储。它能在O(1)的时间复杂度下完成list两端的push和pop操作。但是因为每次操作都需要重新分配ziplist的内存，所以实际复杂度和ziplist的内存使用量相关。

ziplist以下这些属性

```c++
// ziplist结构
<zlbytes><zltail><zllen><entry><entry>...<entry><zlend>
```

* zlbytes 字段的类型是uint32_t, 这个字段中存储的是整个ziplist所占用的内存的字节数
* zltail 字段的类型是uint32_t, 它指的是ziplist中最后一个entry的偏移量. 用于快速定位最后一个entry, 以快速完成pop等操作
* zllen 字段的类型是uint16_t, 它指的是整个ziplit中entry的数量. 这个值只占2bytes（16位）: 如果ziplist中entry的数目小于65535(2的16次方), 那么该字段中存储的就是实际entry的值. 若等于或超过65535, 那么该字段的值固定为65535, 但实际数量需要一个个entry的去遍历所有entry才能得到.
* zlend 是一个终止字节, 其值为全F, 即0xff. ziplist保证任何情况下, 一个entry的首字节都不会是255

```c++
<prevlen><encoding><entry-data>

// prevlen：前一个entry的大小
// encoding: 当前entry的类型和长度
// 在entry中存储的是int类型时，encoding和entry-data会合并在encoding中表示

// 当前一个元素长度小于254（255用于zlend）的时候，prevlen长度为1个字节，值即为前一个entry的长度，如果长度大于等于254的时候，prevlen用5个字节表示，第一字节设置为254，后面4个字节存储一个小端的无符号整型，表示前一个entry的长度；
<prevlen><encoding>

// prevlen用来进行遍历访问
```

![ziplist的entry](/%E6%95%B0%E6%8D%AE%E5%BA%93/assets/ziplist%E7%9A%84entry.png)

ziplist节省内存，但是内存分配频繁

ziplist不预留内存空间, 并且在移除结点后, 也是立即缩容

结点如果扩容, 导致结点占用的内存增长, 并且超过254字节的话, 可能会导致链式反应: 其后一个结点的entry.prevlen需要从一字节扩容至五字节. 最坏情况下, 第一个结点的扩容, 会导致整个ziplist表中的后续所有结点的entry.prevlen字段扩容。这就是ziplist的级联更新。

ziplist 不会预留扩展空间，每次插入一个新的元素就需要调用 realloc 扩展内存, 并可能需要将原有内容拷贝到新地址。

因为级联更新的现象的存在，添加、修改、删除元素操作的复杂度在 O(n) 到 O(n^2) 之间。

当作为list和hash的底层实现时，节点之间没有顺序；当作为zset的底层实现时，节点之间会按照大小顺序排列。

Quicklist

以ziplist为结点的双端链表结构. 宏观上, quicklist是一个链表, 微观上, 链表中的每个结点都是一个ziplist。

### set

底层：hashtable或者intset

拉链法解决哈希冲突。扩容是两倍扩容，缩容也是缩小一倍。

***渐进式hash***

扩容和收缩操作不是一次性、集中式完成的，而是分多次、渐进式完成的。Redis采用渐进式 rehash,这样在进行渐进式rehash期间，字典的删除查找更新等操作可能会在两个哈希表上进行，第一个哈希表没有找到，就会去第二个哈希表上进行查找。但是进行 增加操作，一定是在新的哈希表上进行的。

```c++

typedef struct intset {
    uint32_t encoding; // 表示编码方式，的取值有三个：INTSET_ENC_INT16, INTSET_ENC_INT32, INTSET_ENC_INT64
    uint32_t length; // 代表其中存储的整数的个数
    int8_t contents[]; // 存储数据的buffer，int8_t就可以都支持上面三种类型
} intset;
```

往int16的intset里插入int32的数，会触发类型升级，所有的数都升级成int32。

### hash

底层：hashtable或ziplist

适合用来存储一个完整的对象

### zset

底层：ziplist或skiplist

![skiplist](/%E6%95%B0%E6%8D%AE%E5%BA%93/assets/skiplist.png)

查找时会从最顶层链表的头节点开始遍历，如果当前节点的下一个节点比目标值小，则继续向右查找，否则就向下一层查找。直到找到目标。

zset允许重复的 score 值：多个不同的 member 的 score 值可以相同。所以需要同时对比score和member。

跳表第k层的元素会按一定的概率p在第k+1层出现，这种概率性就是在插入过程中实现的。

插入时首先插入最底层，然后随机计算是否要插入第k层。

每个节点都带有一个高度为 1 层的后退指针，用于从表尾方向向表头方向迭代：当执行 ZREVRANGE 或 ZREVRANGEBYSCORE 这类以逆序处理有序集的命令时，就会用到这个属性。

## 持久化

### RDB快照

***手动触发*** save同步和bgsave异步

RDB Point-to-time snapshot，以二进制文件保存数据库某一时刻所有数据对象的内存快照。

RDB持久化的两个函数：

rdbSave：同步执行，持久化过程会阻塞命令，无法对外提供服务。手动触发

rdbSaveBackground：定时通过fork子进程，子进程会拷贝父进程的内存数据，在子进程进行持久化保存内存对象，父进程数据变化不涉及子进程。

![rdb流程](/%E6%95%B0%E6%8D%AE%E5%BA%93/assets/rdb%E6%B5%81%E7%A8%8B.png)

触发自动执行的场景：

* save m n配置规则自动触发

    save命令就是redis的一个周期性函数，通过配置文件的规则来自动持久化。save m n的意思就是在m秒内有n此写入就触发

* 从节点全量复制时，主节点发送rdb文件给从节点，会触发主节点bgsave
* 执行debug reload命令重新加载redis
* 执行shutdow命令

如果服务器崩溃了，以上次的rdb完整文件做恢复。

## AOF

Aappend only file，redis的重放日志，追加式的日志文件。

每次修改类型的命令输入，就会追加到aof buffer，再写入aof文件。

由于内核缓冲区要等写满才同步到磁盘，可能会有丢数据风险。aof写入缓冲区后有三种策略：

* 每次都fsync同步
* 不刷新缓冲区
* 满足同步条件时调用fsync，理论上只会丢失1s的数据。

redis还有一个选项 no-appendfsync-on-rewrite 表示在AOF重写期间是否禁止调用fsync，默认为no

### AOF 重写

简单的说就是AOF文件大小的压缩，当大小超过阈值时就会开始重写。通过内存数据对象的状态重写一份aof文件。

通过fork子进程，用RDB或者AOF的方式，先写入前半段，父进程通过匿名管道发送增量aof命令，追加到重写的aof buffer中。

## RDB和AOF对比

* RDB是数据快照，比起AOF的日志重放操作，数据恢复更快
* AOF的数据更具有实时性，更新比较频繁。
* RDB持久化过程内存占用大，AOF只是追加，更新快

## 作为缓存时的系列问题

缓存穿透\缓存击穿\缓存雪崩\缓存和数据一致性

### 缓存穿透

一直访问缓存和数据库都没有的数据,导致每次都直接读数据库,给数据库造成IO压力.

解决:

1.从缓存取不到的数据，在数据库中也没有取到，这时也可以将key-value对写为key-null, 过期时间设置较短.
2.布隆过滤器

### 缓存击穿

缓存击穿是指缓存中没有但数据库中有的数据

解决:

1.设置热点数据永远不过期
2.接口限流,熔断,失败快速返回
3.加锁

### 缓存雪崩

缓存雪崩是指缓存中数据大批量到过期时间，而查询数据量巨大.缓存击穿指并发查同一条数据，缓存雪崩是不同数据都过期了，很多数据都查不到从而查数据库。

解决:

1.缓存数据的过期时间设置随机，防止同一时间大量数据过期现象发生。

2.如果缓存数据库是分布式部署，将热点数据均匀分布在不同的缓存数据库中。

3.设置热点数据永远不过期。

## 缓存数据一致性
