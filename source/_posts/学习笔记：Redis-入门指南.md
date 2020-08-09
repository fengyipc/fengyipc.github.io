---
title: 学习笔记：Redis 入门指南
abbrlink: 19531
tags:
  - 学习笔记
  - Redis
  - 数据库
categories: 学习笔记
date: 2020-05-20 18:35:51
abstract: Redis 是 Remote Dictionary Server （远程字典服务器）的缩写，它是一个开源的高性能键值对数据库，提供多种键值数据类型来适应不同的存储需求。借助许多高级接口可以实现如缓存、队列系统等功能。本文主要记录阅读《Redis 入门指南》一书 部分章节时的学习笔记。
---

## 简介

Redis 是 Remote Dictionary Server （远程字典服务器）的缩写，它是一个开源的高性能键值对数据库，提供多种键值数据类型来适应不同的存储需求。借助许多高级接口可以实现如缓存、队列系统等功能。

Redis 支持五种数据类型：

- 字符串类型
- 散列类型
- 列表类型
- 集合类型
- 有序集合类型

Redis 数据库中的所有数据都存储在内存中，读写速度极快。也提供持久化支持，可以将数据异步写入硬盘中，同时不影响继续提供服务。

Redis 可以为每个键设置 TTL ，到期后键自动删除，使其可以作为缓存系统使用。Redis 的列表类型键可以用来实现队列，并且支持阻塞式读取，很容易实现高性能的优先级队列。还支持“发布/订阅”的消息模式。

---

## 数据类型

### 字符串类型 ( String )


字符串类型是 Redis 中最基本的数据类型，可以存储任何形式的字符串，包括二进制数据。字符串类型是其他 4 种数据类型的基础，其他数据类型只是组织字符串的形式不同。

> Redis 键名规范：对象类型 : 对象 ID : 对象属性  
> 多个单词以' . ' 分隔

**命令**

```
赋值 与 取值
SET key value
GET key

递增/减 数字
INCR key
DECR key

增/减 指定整数
INCRBY key increment
DECRBY key decrement

增/减指定浮点数
INCRBYFLOAT key increment
DECRBYFLOAT key decrement

向尾部追加值 (若键不存在，则设置为 value)
APPEND key value

获取字符串长度
STRLEN key

同时获得/设置 多个键值
MGET key [key ...]
MSET key value [key value ...]

位操作
GETBIT key offset
SETBIT key offset value

获取统计的字节范围内，二进制位为1的个数
BITCOUNT key [start] [end]

对多个字符串键进行位运算 支持 AND、OR、XOR、NOT
BITOP operation destkey key [key ...]
```

### 散列类型 ( Hash Table )

散列（ hash ）类型的键值是一种字典结构，其存储了字段和字段值的映射，但字段值只能是字符串。最多容纳 2^32-1 个元素。

散列类型适合存储：以类型和 ID 构成键名，字段表示属性，字段值存储属性值。

![](/images/fig39.png)

**命令**

```
赋值与取值
HSET key field value
HGET key field // 不区分插入和更新 插入返回 1 ，更新返回 0

HMSET key field value [field value ...]
HMGET key field [field ...]

返回所有字段和字段值组成的列表
HGETALL key

存在返回 1 ，否则返回 0 | 键不存在也返回 0
HEXISTS key field

当字段不存在时赋值 返回 1， 存在是不执行任何操作 返回 0 (原子操作)
HSETNX key field value

增加数字
HINCRBY key field increment

删除字段 返回值为删除的个数
HDEL key field [field ...]

获取键中所有字段名
HKEYS key

获取键中所有字段值
HVALS key

获取字段数量
HLEN key
```

### 列表类型 ( List)

列表类型 ( list ) 可以存储一个有序的字符串列表，常用操作是向两端添加元素，或者获得列表的某一片段。内部使用双向链表实现，故获取接近两端的元素很快。同样最多容纳 2^32-1 个元素。

**命令**

```
向列表两端增加元素 返回值为增加后的长度
LPUSH key value [value ...]
RPUSH key value [value ...]

从列表中弹出元素 | 先从列表中移除，返回被移除的元素
LPOP key
RPOP key

获取列表中元素个数
LLEN key

获取元素片段 | 索引从 0 开始，包含首尾元素 | 支持负索引，右边第一个为 -1
LRANGE key start stop

删除列表中指定的值 | count > 0 删除前 count 个 | count < 0 删除后 -count 个 | count = 0 全删 
LREM key count value

获得/设置指定索引的元素值
LINDEX key index
LSET key index value

删除索引范围之外的所有元素
LTRIM key start end

向列表中插入元素 | 在列表中从左向右查找值为 pivot 的元素，根据第二个参数决定将 value 插到前面或后面
LINSERT key BEFORE/AFTER pivote value

将一个元素从一个列表转移到另一个列表 | 原子操作
RPOPLPUSH source destination
```

### 集合类型 ( Set )

集合在 Redis 内部使用值为空的哈希表实现，故加入、删除、判断元素是否存在等操作时间复杂度均为 O(1)。可以方便地在多个集合类型的键之间进行并集、交集和差集运算。

**命令**

```
增加/删除元素 | 返回成功增加/删除的元素个数
SADD key member [member ...]
SREM key member [member ...]

获取集合中的所有元素
SMEMBERS key

判断元素是否在集合里 | 存在返回 1 | 不存在返回 2
SISMEMBER key member

集合间运算 | 返回值为结果
SDIFF  key [key ...] // A - B
SINTER key [key ...] // A & B
SUNION key [key ...] // A | B

获取集合中元素个数
SCARD key

进行集合运算并将结果存储 | 不直接返回结果，而存储至 destination 中
SDIFFSTORE  destination key [key ...]
SINTERSTORE destination key [key ...]
SUNIONSTORE destination key [key ...]

随机获取集合中的元素 | count 为正数，返回 count 个不重复元素 
count > total 返回全部 | count 为负数 返回 -count 个 可重复元素
随机算法为 先随机选择一个哈希桶，再从桶中随机选择一个元素 故元素少的桶中的元素随机中可能性更高
SRANDMEMBER key [count]

随机弹出一个元素
SPOP key
```

### 有序集合类型 ( Sorted Set )

在集合类型的基础上为每个元素关联一个分数，用于排序。使得我们可以完成获得最高/最低分数的前 N 个元素，获得指定分数范围内的元素等与分数相关的操作。

与列表的区别：

- 列表类型通过链表实现，越靠近两端获取速度越快，适合用于实现“新鲜事”或“日志”这样很少访问中间元素的应用。
- 有序集合类型使用哈希表和跳表实现，即使读取位于中间的数据速度也很快 （时间复杂度为 O( log( N ) )。
- 列表不能简单地调整某个元素的位置，但有序集合可以通过更改这个元素的分数实现。
- 有序集合比列表类型更耗费内存。

**命令**

```
增加元素 | 如果该元素存在，就更新分数 | 返回值为新加入的元素个数 | 可使用 +inf -inf 代表正负无穷
ZADD key score member [score member ...]

获得元素的分数
ZSCORE key member

获得排名在某个范围的元素列表 | 加 WITHSCORES 连同分数一起返回
分数相同则按元素字典序 | ZRANGE 从小到大 | ZREVRANGE 从大到小
ZRANGE key start stop [withscores]
ZREVRANGE key start stop [withscores]

获得指定分数范围的元素 | 不希望包括端点值，可在分数前加 "("  | limit offset count 等同 SQL 用法
ZRANGEBYSCORE key min max [withscores] [limit offset count]
ZREVRANGEBYSCORE key min max [withscores] [limit offset count]

增加某个元素的分数 | 返回值为增加后的分数 | 可为负 | 如指定元素不存在，则先建立分数为 0 的元素后执行操作
ZINCRBY key increment member

获得集合中的元素数量
ZCARD key

获得指定分数范围内的元素个数
ZCOUNT key min max

删除一个或多个元素
ZREM key member [member ...]

删除一定排名范围内的元素
ZREMRANGEBYRANK key start stop

删除一定分数范围内的元素
ZREMRANGEBYSCORE key min max

获得元素排名 | ZRANK 从小到大排 | ZREVRANK 从大到小排 | 从 0 开始
ZRANK key member
ZREVRANK key member

计算有序集合的交集存储在 dest 中 | 返回值为 dest 中的元素个数 
sum min max 决定dest中元素分数的计算方式
ZINTERSTORE dest numkeys key [key ...] [WEIGHTS weight [weight ...]] [SUM | MIN | MAX ]

计算并集 同上

ZUNIONSTORE ...
```

---

## 学习资料

> 《Redis入门指南》