---
title: 学习笔记：Redis 设计与实现 —— 数据结构与对象
tags:
  - 学习笔记
  - Redis
  - 数据库
categories: 学习笔记
abbrlink: 63042
date: 2020-07-24 10:39:08
abstract: 《Redis 设计与实现》一书系统而全面地描述了 Redis 内部运行机制，本文主要记录阅读《Redis 设计与实现》一书 数据结构与对象 时的学习笔记。
---

# 简单动态字符串 ( simple dynamic string, SDS )

Redis 使用 SDS 作为默认字符串表示，只有在一些无需修改字符串值，如打印日志的地方才会使用C 字符串。

Redis 在数据库中存储的键值对的键和值，底层实现都是一个SDS。除了用来保存数据库中的字符串值外，SDS 还被用作缓冲区。

---

## SDS 的定义

每个 `sds.h/sdshdr` 结构表示一个 SDS 值：
```C
struct sdshdr {
    // 记录 buf 数组中已使用字节的数量
    // 等于 SDS 所保存字符串的长度 不包括'\n'
    int len;

    // 记录 buf 数组中未使用字节的数量
    int free;

    // 字节数组，用于保存字符串
    char buf[];
};
```

![SDS示例](https://box.kancloud.cn/2015-09-13_55f50d86a66ae.png)

---

## SDS 与 C 字符串的区别

### 常数复杂度获取字符串长度 O ( 1 )

### 杜绝缓存区溢出

当 SDS API 需要对 SDS 进行修改时，API 会自动检查并将 SDS 的空间扩展至执行修改所需的大小。（实际不止）

### 减少修改字符串时带来的内存重分配次数

通过未使用空间， SDS 实现了空间预分配和惰性空间释放两种优化策略。

#### 空间预分配

当需要对 SDS 进行空间扩展时，会为 SDS 分配额外的未使用空间。

分配策略：

- `len` 小于 `1 MB` 时，分配 `free = len` 的未使用空间
- `len` 大于 `1 MB` 时，分配 `free = 1 MB` 的未使用空间

通过这种预分配策略， SDS 将连续增长 `N` 次字符串所需的内存重分配次数从必定 `N` 次降低为最多 `N` 次。

#### 惰性空间释放

当缩短字符串时，不立即回收多出来的空间，而是使用 `free` 属性将其记录下来，并等待将来使用。同时 SDS 也提供相应的 API 供手动释放 SDS 未使用的空间，避免内存浪费。

### 二进制安全

由于使用 `len` 属性保留长度，不需要 `\0` 判断，没有存储限制，故 SDS 是二进制安全的。

### 兼容部分 C 字符串函数

虽然 SDS 是二进制安全的，但依旧使用'\n' 作为字符结尾，就是为了可以让 SDS 重用一部分 `string.h` 中的函数，如 `strcasecmp` 、 `strcat` 等函数。

### 总结

|C 字符串	| SDS | 
|  :-----   |  :-----   |
|获取字符串长度的复杂度为 O(N) 。	|获取字符串长度的复杂度为 O(1) 。|
API 是不安全的，可能会造成缓冲区溢出。	|API 是安全的，不会造成缓冲区溢出。|
修改字符串长度 N 次必然需要执行 N 次内存重分配。	|修改字符串长度 N 次最多需要执行 N 次内存重分配。|
只能保存文本数据。	|可以保存文本或者二进制数据。|
可以使用所有 <string.h> 库中的函数。	|可以使用一部分 <string.h> 库中的函数。|

---

# 链表

链表是 Redis 列表键的底层实现之一：当一个列表键含有数量较多的元素，或者列表中包含的元素都是长字符串时，Redis 会使用链表作为列表键的底层实现。

除了链表键之外， 发布与订阅、慢查询、监视器等功能也用到了链表， Redis 服务器本身还使用链表来保存多个客户端的状态信息， 以及使用链表来构建客户端输出缓冲区（output buffer）。

## 链表和链表节点的实现

每个链表节点使用一个 `adlist.h/listNode` 结构来表示：

```C
typedef struct listNode {

    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;

} listNode;
```

实际链表，使用 `adlist.h/list` 来表示

```C
typedef struct list {

    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 链表所包含的节点数量
    unsigned long len;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);

} list;
```

![链表结构](https://box.kancloud.cn/2015-09-13_55f50fb39b6cb.png)

Redis 的链表实现的特性可以总结如下：

- 双端： 链表节点带有 `prev` 和 `next` 指针， 获取某个节点的前置节点和后置节点的复杂度都是 O(1) 。
- 无环： 表头节点的 `prev` 指针和表尾节点的 `next` 指针都指向 NULL 。
- 带表头指针和表尾指针： 获取链表的表头节点和表尾节点的复杂度为 O(1) 。
- 带链表长度计数器： 获取链表中节点数量的复杂度为 O(1)。
- 多态： 链表节点使用 `void*` 指针来保存节点值， 通过为链表设置不同的类型特定函数， Redis 的链表可以用于保存各种不同类型的值。

# 字典

Redis 的字典使用哈希表作为底层实现。

## 字典的实现

### 哈希表

哈希表由 dict.h/dictht 结构定义：

```C
typedef struct dictht {

    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht;
```

### 哈希表节点

哈希表节点使用 `dictEntry` 结构表示， 每个 `dictEntry` 结构都保存着一个键值对：

```C
typedef struct dictEntry {

    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;

} dictEntry;
```

### 字典

Redis 中的字典由 dict.h/dict 结构表示：

```C
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

} dict;
```

`type` 属性和 `privdata` 属性是针对不同类型的键值对， 为创建多态字典而设置的：

- `type` 属性是一个指向 `dictType` 结构的指针， 每个 `dictType` 结构保存了一簇用于操作特定类型键值对的函数， Redis 会为用途不同的字典设置不同的类型特定函数。
- 而 `privdata` 属性则保存了需要传给那些类型特定函数的可选参数。
- `ht` 属性是一个包含两个项的数组， 数组中的每个项都是一个 `dictht` 哈希表， 一般情况下， 字典只使用 `ht[0]` 哈希表， `ht[1]` 哈希表只会在对 `ht[0]` 哈希表进行 `rehash` 时使用。

```C
typedef struct dictType {

    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);

    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);

    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);

    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);

    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);

} dictType;
```

![普通状态下的字典](https://box.kancloud.cn/2015-09-13_55f5120772706.png)

## 哈希算法

Redis 计算哈希值和索引值的方法如下：

```C
# 使用字典设置的哈希函数，计算键 key 的哈希值
hash = dict->type->hashFunction(key);

# 使用哈希表的 sizemask 属性和哈希值，计算出索引值
# 根据情况不同， ht[x] 可以是 ht[0] 或者 ht[1]
index = hash & dict->ht[x].sizemask;
```

当字典被用作数据库的底层实现， 或者哈希键的底层实现时， Redis 使用 MurmurHash2 算法来计算键的哈希值。

MurmurHash 算法最初由 Austin Appleby 于 2008 年发明， 这种算法的优点在于， 即使输入的键是有规律的， 算法仍能给出一个很好的随机分布性， 并且算法的计算速度也非常快。

## 解决键冲突

Redis 的哈希表使用链地址法（separate chaining）来解决键冲突： 每个哈希表节点都有一个 `next` 指针， 多个哈希表节点可以用 `next` 指针构成一个单向链表， 被分配到同一个索引上的多个节点可以用这个单向链表连接起来， 这就解决了键冲突的问题。

处于速度考虑，新节点总是添加在表头位置，访问世间复杂度 O ( 1 )。

## Rehash

> 哈希表负载因子（load factor）定义为：α= 填入表中的元素个数 / 哈希表的长度

为了让哈希表的负载因子维持在一个合理的范围之内， 当哈希表保存的键值对数量太多或者太少时， 程序需要对哈希表的大小进行相应的扩展或者收缩。


# 笔记

> 笔记做到这里，深深感受到此书之精妙，我努力试图从其中摘选出最精华的部分，但越写越想把整本书都记录下来。于是才明白，这一作者反复打磨、精炼的产物，以我的水平还不足以去轻视其中任何一句，故选择不再做笔记，当需要时再阅读原书。
