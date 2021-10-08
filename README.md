# pikafs
polarfs 开源实现

# 介绍
[pika](https://github.com/OpenAtomFoundation/pika) 是一个提供redis接口持久化数据的KV数据库。有效的解决了单机情况的高成本的内存使用问题。随着pika开源后在互联网公司的广泛使用，pika面临新的挑战。核心痛点在于pika无法有效解决单数据库容量严重依赖于单块硬盘容量的问题和基础设施逐渐完善，如何有效使用云原生环境进一步降低DBA同学的负担。

## 痛点

为了解决这个痛点，pika在做sharing nothing的集群方案，在开发这个系统的过程中遇到了架构导致的问题。

![share nothing](https://github.com/kernelai/pikafs/blob/main/doc/pic/share_nothing_argchitecture.png)

例如：

* 由于按照hash的方式来对数据切分后，无法对数据进行全局scan的问题。
* hash的分布不能有效解决key热点的问题。我们在维护codis-pika 集群的时候明显能发现这个问题。
* 分布式事务的支持更是遥遥无期。
* hash的slot数目需要在创建表的时候就确定下来，后期无法修改。不管业务和DBA同学都无法快速有效的准确估算出未来集群可以容纳多少数据。因此实际操作往往会过大的设置slot数目。
* pika 中是slot数目对应一个rocksdb实例，这样是为了方便做数据同步。但slot数目设置过大会导致rocksdb 过多的占用内存资源。例如对于一个100个slot的pika实例，仅仅memtable的内存占用可能会超过10G,这部分资源很大一部分可能是浪费的。因为rocksdb会对memtable进行预分配内存提高性能。尽管可以通过arena参数减少预分配的大小。但arena与slot数目、写性能强绑定，如何根据业务场景进行参数配置对于DBA同学也是个负担。此外较多slot数目增加了数据同步、主从保活的负担。这点我们在维护codis-pika集群中深有体会。
* 分布式方案multi raft保证数据一致性。slot数目较多对于元数据管理也会相对复制。
* pika集群可以支持多table。通过table进行了数据隔离，但目前的现状是没有进行按照table服务质量管理。


## 契机
当我们看到AWS aurora 和阿里云的PorlarDB时，似乎看到了一丝曙光。尽管它是应用于SQL数据库，我们大胆的设想，这种架构能否应用于KV数据库呢？

![polardb](https://github.com/kernelai/pikafs/blob/main/doc/pic/polardb.png)
存储和计算分离的架构带来一系列的优势：

![share storage](https://github.com/kernelai/pikafs/blob/main/doc/pic/share_storage_argchitecture.png)

* 在基础设施逐步完善的大环境下，天生适合云原生。计算资源就成为无状态的服务，借助于K8s环境按照业务需求快速的扩缩容。把DBA同学要深入理解数据库的使命，转换为仅仅面向资源。据悉阿里云维护数千数polardb据库，仅需1~2人的人力。
* 一写多读的架构使得面对写节点故障时主从切换变得异常简单，不需要考虑内部各个slot的主从状态。
* 底层依赖分布式式存储，volume从10G~100T可以动态扩容。节点存储容量不再是核心痛点。
* 计算层不需要对数据进行切片，完美解决全局scan的问题，同时也为pika支持事务的能力扫清了障碍。
* 计算节点面对读热点，只需水平扩展读节点就可以了。甚至借助k8s的能力自动进行计算能力的扩缩容。
* 尽管底层的存储数据也依赖于raft保持数据一致性，存储multi raft的情况。但切片的力度较大，raft group的数据较少。
* 支持多volume，可以进行volume的隔离，同时也可以针对volume进行io隔离，隔离级别可以深入的更底层。
* 计算节点也不需要创建更多的slot。导致资源的浪费。
* 同时可以进一步较少pika的锁机制，提升并发读。每次请求会请求一次锁，来查看table的状态。不写binlog，仅依靠rocksdb的wal进行数据同步。
* 在面对多读（如10读）的情况下，进一步减少数据的拷贝。因为数据在分布式存储的上层看起来只存储了一份。
* 对存储资源、内存资源、计算资源进一步池化，规模效应下进一步提高资源利用率。

## 银弹？
1. 无法简单解决写节点的水平扩展。当遇到需要高写入的场景，无法进行水平扩展。据说aws和阿里云都在做这方面的尝试，pika 也可以亦步亦趋。

2. 另一大痛点在于无法在开源社区里找到一份高性能低延迟的分布式文件系统方案。阿里为此开发了针对关键业务如数据库的polarfs。可惜的是polardb for pg 开源了，但polarfs并没有开源。通用性的存储cephfs未必能满足低延迟的需求。

3. pika团队对rocksdb的代码理解还不够深入。或者说理解深入的前辈去了阿里做polardb。

## roadmap

做存储与计算分离的过程是可以多路并行开发。
1. 满足读写分离的高性能proxy, 按照key 把请求打散到多个只读节点，进一步降低sst文件的缓存占用。
2. rocksdb 存储引擎针对分布式文件存储的性能优化。这点相对清晰，可以借助rocksdb中的env 做些改造工作。
3. rocksdb 在分布式存储的环境下针对一写多读的场景的改造。其中核心的几个问题：
 * 只读节点如何根据wal重建memtable, 通过wal record？

 * wal文件能否拆分，进一步充分利用高并发，高延迟的分布式文件系统。wal 拆分后如何处理故障后根据wal重建数据。

 * 在哪个节点进行compact操作。

 * compact的过程中会检查key的可见性。例如一个读请求在只读节点，scan操作，扫描全局key，比较慢。此时到来一个删除操作的写请求。但读请求根据snapshot是可以看到过期的key。这个key是不能真正删除的。不同节点如何做compact？问题变为如何知道key的全局引用情况?

 * 计算节点、proxy的operator，实现运维自动化

    

4. 做一款非通用只针对数据库场景的性能接近本地ssd盘的分布式文件系统 -- pikafs？充分利用cpu绑核、非阻塞线程模型、SPDK、RDMA、OS-bypass、zero-copy等技术实现高性能、低延迟。
