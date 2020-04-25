[TOC]

# Redis zset底层数据结构 

## #1 什么是zset

zset数据类型的对象是有序的,zset中的每一个元素都有一个score作为排序的依据 


## #2 zset底层数据结构

zset底层数据结构有两种: 压缩列表(`ziplist`) 或者 跳跃列表(`skiplist`)

这两种数据结构要么是`ziplist`,要么是`skiplist`,不会两种数据结构同时存在,那么什么时候是`ziplist`,什么时候是`skiplist` ?

==同时==满足以下两个条件时,zset对象使用 `ziplist` : 

- 保存的元素数量小于128个
- 保存的所有元素长度都小于64字节

## #3 压缩列表(`ziplist`) 

==同时==满足以下两个条件时,zset对象使用 `ziplist` : 

- 保存的元素数量小于128个
- 保存的所有元素长度都小于64字节

> 为什么需要压缩列表 ? 

- 在集合元素较少，元素类型较小时，使用压缩列表，这样由于数据元素较少，故虽然压缩列表效率较低，但是基本不会有多大影响，而且可以充分利用压缩列表的连续内存和紧凑的数据结构实现来节省内存。

### #3.1 什么是压缩列表 ? 

每一个元素占用两个相邻的节点保存数据,一个存的是数据值(value),另一个存的是权值(score),并且压缩列表内的集合元素按分值从小到大的顺序进行排列，小的放置在靠近表头的位置，大的放置在靠近表尾的位置。



## #4 跳跃列表(`skiplist`)

> 一个 zset 数据结构同时包含一个字典(`dict`)和一个跳跃表(`skiplist`)


```c++
/*
 * zset
 */
typedef struct zset {

    // 字典，键为成员，值为分值
    // 用于支持 O(1) 复杂度的按成员取分值操作
    dict *dict;

    // 跳跃表，按分值排序成员
    // 用于支持平均复杂度为 O(log N) 的按分值定位成员操作
    // 以及范围操作
    zskiplist *zsl;

} zset;
```


```
/*
 * 跳跃表zskiplist
 */
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;

} zskiplist;
```


