#sds
Simple Dynamic String，简单动态字符串.
Redis 的字符串表示还应该是二进制安全的： 程序不应对字符串里面保存的数据做任何假设
```
struct sdshdr {

    // buf 已占用长度
    int len;

    // buf 剩余可用长度
    int free;

    // 实际保存字符串数据的地方
    char buf[];
};
```
# 双端链表
 RPUSH 、 LPOP , LLEN , LRANGE 
Redis 列表使用两种数据结构作为底层实现：
* 双端链表
* 压缩列表
因为双端链表占用的内存比压缩列表要多， 所以当创建新的列表键时， 列表会优先考虑使用压缩列表作为底层实现， 并且在有需要的时候， 才从压缩列表实现转换到双端链表实现。
数据结构：
```
typedef struct listNode {

    // 前驱节点
    struct listNode *prev;

    // 后继节点
    struct listNode *next;

    // 值
    void *value;

} listNode;

typedef struct list {

    // 表头指针
    listNode *head;

    // 表尾指针
    listNode *tail;

    // 节点数量
    unsigned long len;
} list;
```

# dict
用途：

* 实现数据库键空间（key space）；
* 用作 Hash 类型键的底层实现之一；

渐进式 rehash
字典的收缩

```
/*
 * 字典
 *
 * 每个字典使用两个哈希表，用于实现渐进式 rehash
 */
typedef struct dict {

    // 特定于类型的处理函数
    dictType *type;

    // 类型处理函数的私有数据
    void *privdata;

    // 哈希表（2 个）
    dictht ht[2];

    // 记录 rehash 进度的标志，值为 -1 表示 rehash 未进行
    int rehashidx;

    // 当前正在运作的安全迭代器数量
    int iterators;

} dict;

/*
 * 哈希表
 */
typedef struct dictht {

    // 哈希表节点指针数组（俗称桶，bucket）
    dictEntry **table;

    // 指针数组的大小
    unsigned long size;

    // 指针数组的长度掩码，用于计算索引值
    unsigned long sizemask;

    // 哈希表现有的节点数量
    unsigned long used;

} dictht;
```

# skiplist
和字典、链表或者字符串这几种在 Redis 中大量使用的数据结构不同， 跳跃表在 Redis 的唯一作用， 就是实现有序集数据类型。
跳跃表将指向有序集的 score 值和 member 域的指针作为元素， 并以 score 值为索引， 对有序集元素进行排序

# 内存映射数据结构
虽然内部数据结构非常强大， 但是创建一系列完整的数据结构本身也是一件相当耗费内存的工作， 当一个对象包含的元素数量并不多， 或者元素本身的体积并不大时， 使用代价高昂的内部数据结构并不是最好的办法。
为了解决这一问题， Redis 在条件允许的情况下， 会使用内存映射数据结构来代替内部数据结构。
内存映射数据结构是一系列经过特殊编码的字节序列， 创建它们所消耗的内存通常比作用类似的内部数据结构要少得多， 如果使用得当， 内存映射数据结构可以为用户节省大量的内存。
不过， 因为内存映射数据结构的编码和操作方式要比内部数据结构要复杂得多， 所以内存映射数据结构所占用的 CPU 时间会比作用类似的内部数据结构要多

# 压缩列表 ziplist

# 数据结构
## 字符串
REDIS_STRING
两种编码方式：
* REDIS_ENCODING_INT 使用 long 类型来保存 long 类型值。
* REDIS_ENCODING_RAW 则使用 sdshdr 结构来保存 sds （也即是 char* )、 long long 、 double 和 long double 类型值。
命令：SET GET

## 哈希表 
REDIS_HASH
命令：HSET HGET
实现方式：
* REDIS_ENCODING_ZIPLIST key 和 value 顺序被压入 ziplist，创建新哈希时，默认采用
* REDIS_ENCODING_HT 和链表相同的升级方式

## 链表
REDIS_LIST
命令：LPUSH 、 LRANGE 等
实现方式：
* REDIS_ENCODING_ZIPLIST 创建新链表，默认使用
* REDIS_ENCODING_LINKEDLIST 当达到两个条件：从ziplist转化为当前实现
** 向列表中add一个过长的字符串（64）
** 链表长度超过一个值（512个）

阻塞命令：BLPOP 、 BRPOP 和 BRPOPLPUSH ，当链表为空时，阻塞客户端
server.db[i]->blocking_keys来实现，它是一个dict。

## 集合
REDIS_SET
命令：SADD 、 SRANDMEMBER
实现方式：
* REDIS_ENCODING_INTSET 如果插入的第一个元素为 long long，采用此模式
* REDIS_ENCODING_HT 1， 如果插入第一个元素部位 long long 2， 元素个数超过512 3，试图插入一个非 long long



阻塞解除：当list中被插入元素 or 超时 or 连接被终止

阻塞超时：每次 Redis 服务器常规操作函数（server cron job）执行时， 程序都会检查所有连接到服务器的客户端， 查看那些处于“正在阻塞”状态的客户端的最大阻塞时限是否已经过期， 如果是的话， 就给客户端返回一个空白回复， 然后撤销对客户端的阻塞




# PUB/SUB

Redis 通过 PUBLISH 、 SUBSCRIBE 等命令实现了订阅与发布模式

struct redisServer {
    // ...
    dict *pubsub_channels;
    // ...
};
