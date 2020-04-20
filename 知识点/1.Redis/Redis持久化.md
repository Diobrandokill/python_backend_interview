[TOC]


# Redis 持久化


## #1 什么是Redis持久化

Redis是Cache级别的缓存,数据存在内存中,如果突然掉电或者宕机,数据将会全部丢失,所以需要将Redis的数据备份到硬盘中永久保存


## #2 持久化方案

目前Redis持久化方案主要有两种: RDB和AOF

> RDB: 快照

快照是一次全量备份

> AOF: 日志

AOF日志是持续的增量备份,AOF日志在长期的运行过程中会变得无比的巨大,数据库重启时需要加载AOF日志进行命令重放,所以需要定期对AOF重写,给AOF日志进行瘦身


## #3 Redis持久化方案--RDB

![20200305192944-image.png](https://raw.githubusercontent.com/Coxhuang/yosoro/master/20200305192944-image.png)


Redis是单线程程序,这个线程要同时负责多个客户端的并发读写操作,同时还需要进行进行内存快照,内存快照要求Redis必须进行文件I/O操作,可文件I/O操作不能使用多路复用API,这时Redis就需要一边持久化,一边响应客户端的请求,持久化的同时,内存数据结构还在改变,那么就存在一个问题: 一个大型的hash数据正在持久化,这时,客户端请求是要把这个hash数据删掉,可是还没持久化完,Redis应该如何处理??? 

Redis使用系统的多进程COW(Copy On Write)机制来实现快照持久化,Redis在做持久化时会fork一个子进程,该子进程做数据持久化,不会修改现有的内存数据结构,它只是对数据结构进行遍历读取,然后序列化写到磁盘中,但是父进程不一样,它必须持续接受客户端的请求,然后对内存数据结构进行不间断的修改

### #3.1 RDB配置


```
# 时间策略
save 900 1
save 300 10
save 60 10000

# 文件名称
dbfilename dump.rdb

# 文件保存路径
dir /home/work/app/redis/data/

# 如果持久化出错，主进程是否停止写入
stop-writes-on-bgsave-error yes

# 是否压缩
rdbcompression yes

# 导入时是否检查
rdbchecksum yes
```

- save 900 1 表示900s内如果有1条是写入命令，就触发产生一次快照，可以理解为就进行一次备份
- save 300 10 表示300s内有10条写入，就产生快照
- stop-writes-on-bgsave-error yes 这个配置也是非常重要的一项配置，这是当备份进程出错时，主进程就停止接受新的写入操作，是为了保护持久化的数据一致性问题。如果自己的业务有完善的监控系统，可以禁止此项配置， 否则请开启。
- 关于压缩的配置 rdbcompression yes ，建议没有必要开启，毕竟Redis本身就属于CPU密集型服务器，再开启压缩会带来更多的CPU消耗，相比硬盘成本，CPU更值钱。
- 如果需要禁用RDB配置，只需要在save的最后一行写上：save ""



## #4 Redis持久化方案--AOF

![20200305193005-image.png](https://raw.githubusercontent.com/Coxhuang/yosoro/master/20200305193005-image.png)


AOF只记录对内存进行修改的指令,Redis会在收到客户端修改指令后,进行参数校验,逻辑处理,如果没问题,就立即将该指令文本存储到AOF日志中

### #4.1 AOF配置


```
# 是否开启aof
appendonly yes

# 文件名称
appendfilename "appendonly.aof"

# 同步方式
appendfsync everysec

# aof重写期间是否同步
no-appendfsync-on-rewrite no

# 重写触发配置
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 加载aof时如果有错如何处理
aof-load-truncated yes

# 文件重写策略
aof-rewrite-incremental-fsync yes
```

- appendfsync everysec 它其实有三种模式:

1. always：把每个写命令都立即同步到aof，很慢，但是很安全
2. everysec：每秒同步一次，是折中方案
3. no：redis不处理交给OS来处理，非常快，但是也最不安全

> 一般情况下都采用 everysec 配置，这样可以兼顾速度与安全，最多损失1s的数据。

- aof-load-truncated yes 如果该配置启用，在加载时发现aof尾部不正确是，会向客户端写入一个log，但是会继续执行，如果设置为 no ，发现错误就会停止，必须修复后才能重新加载。


## #5 从持久化中恢复数据

> 数据的备份、持久化做完了，我们如何从这些持久化文件中恢复数据呢？如果一台服务器上有既有RDB文件，又有AOF文件，该加载谁呢？

![20200305193119-image.png](https://raw.githubusercontent.com/Coxhuang/yosoro/master/20200305193119-image.png)

启动时会先检查AOF文件是否存在，如果不存在就尝试加载RDB。那么为什么会优先加载AOF呢？因为AOF保存的数据更完整，通过上面的分析我们知道AOF基本上最多损失1s的数据。


## #6 RDB和AOF对比

- RDB优点：全量数据快照，文件小，恢复快
- RDB缺点：无法保存最近一次快照之后的数据，数据量大会由于I/O严重影响性能
- AOF优点：可读性搞，适合保存增量数据，数据不一丢失
- AOF缺点：文件体积大，恢复时间长


## #7 Redis4.0 混合持久化

重启Redis时,我们很少使用RDB来恢复数据,因为会丢失大量的数据(间隔性备份),我们通常使用AOF日志重放,但是AOF日志相对于使用RDB来说又太慢了,这样子Redis实例很大的时候,启动需要花费很长的时间

Redis4.0为了解决这个问题,带来了一个新的持久化方案--混合持久化

Redis重启的时候,可以现在加RDB的内容,然后在重放增量AOF日志,就可以完全替代之前的AOF全量日志重放,重启效率大幅提升







