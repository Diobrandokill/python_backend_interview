[TOC]

# Redis集群

## #1 什么是Redis集群

将众多小内存的Redis实例整合起来,将分布在多台机器上的众多CPU核心的计算能力聚集到一起,完成海量数据存储和高并发多写操作

## #2 Redis集群方案有哪些? 

> 主要方案有以下两个

- Codis
- Cluster


> Codis

国产开源Redis集群方案


> Cluster

官方提供的Redis集群方案


## #3 Codis

### #3.1 Codis集群方案图

> Codis是无状态的,它只是一个转发代理的中间件,这意味着我们可以启动多个Codis实例,供客户端使用,每个Codis节点是平等的


![20200305175329-image.png](https://raw.githubusercontent.com/Coxhuang/yosoro/master/20200305175329-image.png)


### #3.2 Codis分片原理

Codis主要是将特定的key转发到特定的Redis实例,具体转发如下:

1. Codis默认将所有的key划分到1024个槽位,它首先对客户端传来的key进行CRC32运算计算hash值,再将hash后的整数值对1024求余,这个余数就是对应的槽位
2. 每个槽位对应一个Redis实例,一个Redis实例对应多个槽位


### #3.3 不同的Codis实例之间槽位关系如何同步 ?

==在扩容/删除Redis节点的时候,Codis槽位与Redis对应关系会发生改变,那如何实现Codis同步呢 ???==

Codis需要使用一个分布式同步工具来配置存储数据库实现持久化槽位关系,Codis开始使用Zookeeper,后来etcd也同样支持

Codis槽位关系存储在zookeeper中(并且提供一个Dashboard可以用来观察和修改槽位关系),当槽位发生变化时,Codis Proxy会监听到变化并重新同步槽位关系


### #3.3 新增Redis实例

刚开始Codis只有一个Redis实例的时候,1024个槽位全部指向同一个Redis实例,然后新增一个Redis实例,这时,需要对槽位关系进行修改,将一半的槽位划分到新的Redis节点,这就意味着对这一半的槽位里面的key进行迁移,迁移到新的Redis实例


### #3.4 自动均衡

Redis新增实例,手动均衡太繁琐,所以Codis提供自动均衡功能,自动均衡功能会在系统比较空闲的时候,观察诶个Redis实例对应的槽位数量,如果不均衡,就会自动迁移


### #3.5 Codis优点 

- 设计方案简单,将分布式问题交给第三方插件(zookeeper或etcd)


### #3.6 Codis缺点

- Codis增加了Proxy作为中转层,在数据传输上要比单个Redis开销大
- 相对于单实例Redis,集群的key分布在不同的Redis中,所以就不在支持事务,因为事务只能在单实例中完成
- Codis不是官方项目,Redis更新功能时,Codis更新会滞后



## #4 Redis Cluster


### #4.1 Redis Cluster集群方案图

> Redis Cluster方案中,所有的Redis节点组成一个完全图,任意节点到其他节点都是可达的,去中心化,没有主节点概念

![20200305175525-image.png](https://raw.githubusercontent.com/Coxhuang/yosoro/master/20200305175525-image.png)


### #4.2 Redis Cluster分片原理

Redis Cluster将所有的数据分为2的14次方(16384)槽位,每个Redis节点负责其中的一部分槽位,槽位信息存储在节点中,不像Codis,不需要另外的分布式存储空间存储节点槽位信息,当Redis Cluster的客户端来连接集群时,也会得到一份集群的槽位配置信息,这样客户端要查找某个key,可以直接定位到目标Redis节点


#### #4.2.1 槽位定位算法

Redis Cluster默认会对key使用CRC16算法进行hash,等到一个整数值,然后用这个整数值对16384求余得到具体的槽位信息










