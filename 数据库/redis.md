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

有点类似stl的string库。

可以用来做缓存、session共享

### list

底层为ZipList，一个特殊编码的双向链表。可以存储字符串或者整数，存储整数时是采用整数的二进制而不是字符串形式存储

ziplist以下这些属性

* zlbytes 字段的类型是uint32_t, 这个字段中存储的是整个ziplist所占用的内存的字节数
* zltail 字段的类型是uint32_t, 它指的是ziplist中最后一个entry的偏移量. 用于快速定位最后一个entry, 以快速完成pop等操作
* zllen 字段的类型是uint16_t, 它指的是整个ziplit中entry的数量. 这个值只占2bytes（16位）: 如果ziplist中entry的数目小于65535(2的16次方), 那么该字段中存储的就是实际entry的值. 若等于或超过65535, 那么该字段的值固定为65535, 但实际数量需要一个个entry的去遍历所有entry才能得到.
* zlend 是一个终止字节, 其值为全F, 即0xff. ziplist保证任何情况下, 一个entry的首字节都不会是255

### set

哈希表实现的无序集合。

### hash

适合用来存储一个完整的对象

### zset

两种数据结构实现：压缩列表和跳表。
