---
layout: post
title:  Google File System笔记
description:
categories:
- Tech
tag: Tech
permalink: gfs
---
GFS, Google File System, 传说中的高富帅. Paper最早发表于2003的ACM SIGOPS. 原文地址在这里[The Google File System][gfs]. 简而言之, Google为自己的大数据存开发了一个符合自己需求的, 密集应用型, 可伸缩的分布式文件系统. 整个文件系统运行在大量的廉价机器上, 但依然提供了强劲的灾难冗余能力, 为大量应用和服务提供了高性能的后端. 下面是阅读paper总结的一些GFS主要思想, 一些细节没有写全...
<!--more-->	


###**Overview -- 总览**
首先, paper做了以下说明, 对于Google的文件系统


* 文件通常都很大, 很少处理KB级别的数据
* 文件的修改通常是在尾部追加, 而不是复写. GFS主要提供原子性的追加写操作
* 请求通常分为两类, 大的流数据读取, 小的随机文件读取
* 高带宽的需求优先于低延迟, 所以首要是确保高效的数据处理

简而言之, GFS的操作也是增删改查, 除此外, 提供对一个文件树的*snapshot*功能, 和*record append*, 使得高并发的写入无需使用分布锁. 整个并发写的模式可以理解为producer-consumer queue + 结果的多路归并.

###**系统架构**
![gfs arc][gfs_arc] 

整个Architecture由一个master和多个数据服务器(chunkserver)构成. 文件被分割成固定大小的chunk, 在GFS中这个大小被设为64MB, 并在创建时由master给予一个不可变的全局64位id. 这些chunk被以本地文件的形式存储, 并且冗余数默认为3.

Master负责存储所有的文件metadata, 主要包括命名空间, 访问权限, 文件到chunk的映射, chunk的位置; 除此之外负责chunk的gc, lease, mitigation等操作. 由于文件通常都很大, 所以没有额外设计cache层, 基本只靠linux系统自带的cache..

####**Chunk size**
Chunk size为64MB. 这个是一个文件块的大小. 每一个文件块以linux文本文件格式存储. 选择如此之大的文件块大小可以


* 减少与master的交互, 因为一次对一个文件块的请求就可以在未来读取64MB的数据
* 因为文件块较大, 客户端经常会需要在同一块上执行增删改查等各种操作, 这时可以利用TCP的长连接来减少网络交通
* 减少了在master上metadata的数量, 这样master可以把metadata存在内存里  


不过这样的设计也不是完全有利无弊的, 因为小文件也将占据一整个64MB的chunk. 当一个chunkserver上有很多这样的小文件时, 有可能成为访问热点. 将metadata存在内存里得以让master在后台可以进行周期性的快速扫描, 进行chunk garbage collection, chunk备份, 和chunk负载均衡目前每一个chunk的metadata是64B. 内存存储使内存大小成为了系统瓶颈, 但论文中说目前这在Google并不是一个问题.

####**Metadata**
Metadata是Master存储的与文件相关的描述信息


* Namespace -- 命名空间
* 文件到chunk的映射
* Chunk备份的地址

前两者通过多备份的*operation log*来存储. *Operation log*存储了重要的metadata变动, 这也为并发的操作提供了一个可靠稳定的时间线记录. 这个log也被用以故障回复, 其中设有大量的checkpoint, 以B树的形式保存.
Master并不长期存储每一个文件chunk的地址, 而是在master启动或chunkserver加入的时候通信得知(polling). 这种HeartBeat消息模式很好地避开了复杂的sync模式.


###**Consistencyh Model -- 一致性模型**
GFS确保文件操作的原子性, 并发的成功写入保证consistency, 但并不defined. Defined指在一个操作后可以立刻看到这个操作带来的全部变化. Chunk通过版本号来检测未sync到最新的文件, 并保证不会发送非最新的数据块到客户端.


###**System interaction -- 系统交互**
修改文件时, 对于同一个文件的多个冗余, master会将一个lease, 可以理解为租赁, 交给其中一个chunk, 称改chunk为primary. 它的任务是选择即将进行的一系列文件修改操作的顺序, 并且其他冗余必须遵循这个顺序. 这样就保证了冗余的一致性. 对于比较大的或chunk边缘的写操作, GFS会将之拆分成许多小操作再依次执行. 

整个文件读写的流程如下所示
![gfs write][gfs_wri]

	
1. 客户端向master请求文件对应的primary chunkserver
2. master回复对应的primary的id和索引. 客户端会为将来的读写操作cache这些内容
3. 客户端将数据推送至所有的冗余. 每个chunkserver都会把数据存在一个内部的LRU buffer cache直到数据被使用或是超时
4. 当所有冗余都回复接受完毕, 客户端将写请求发至primary. Primary会给接收到的一些列修改操作依次编号, 并对本地数据进行修改
5. Primary将发送写请求到所有二级冗余, 它们也将依照编号依次执行相同的修改操作
6. 二级冗余回复primary告知操作完成
7. Primary回复给客户端. 在此期间任何发送的错误都会被汇报给客户端, 并由master对应处理

数据流和控制信息流是分开的. 控制信息流将先抵达primary, 再到所有的冗余. 而数据流则是根据距离由近到远地传递. 也就是说, 每次转发都会取最近的冗余节点. 对于Google的网络拓扑, 他们可以通过内部的ip地址来判断距离.

写入的主要操作由*record append*提供, 利用原子操作, 避免了传统读写在这种情况下对一个分布式锁的需求. 整个过程任由primary主导. 

*snapshot*操作提供对一个文件或文件树的快速复制功能. 采用copy-on-write机制, 由master存储同一个chunk的引用计数, 并在必要的时候完成复制.


###**Master Operation -- Master操作**
简而言之, master负责所有命名空间相关的操作, 并负责所有冗余管理.

由于*snapshot*等操作执行时间较长, 为了防止阻塞其他的master操作, 我们利用对不同的命名空间进行lock来保证并发操作的序列化. GFS并没有使用类似于Linux一样的文件夹数据结构, 而是对每个文件保存了完整路径到metadata的映射, 并利用前缀压缩将他们存储在内存里. 任何一个master操作需要先请求一系列的lock. 比如现在要对/a/b/c/d文件进行修改, 我们将请求/a, /a/b, /a/b/c的read lock和/a/b/c/d的write lock. 对同一个lock的不同请求将被序列化.

GFS创建数据冗余主要出于三个原因


* 新建chunk, 主要在创建新文件时. 对应宿主的选择需要考虑当前集群的网络流量, 空间, 热点等情况. 一般情况下, 紧随新建chunk而来的将会是大量的写入操作
* 重新复制, 主要在当冗余达不到需求之时. 比如一部分冗余的宿主故障了
* 负载均衡, 主要为了使网络流量在集群内平衡均匀. master将周期性进行检索操作, 以尽量避免个别host需要处理过度多的请求

###**Garbage Collection -- 垃圾收集**
当一个文件被删除时, GFS并不会立刻释放对应的磁盘空间, 而是标记一个删除时间戳. 在一定时间后, master会在周期性的检索里发现被删除的文件并释放对应空间. 这个时间间隔在GFS里默认为3天. 其间文件可以恢复, 也可以进行强制性删除. 这样的好处在于简化了删除时的信息交流, 并且避免了因GC繁忙而阻塞master正常操作的可能. 此外, 对于未sync而过期的chunk而言, master也会在周期性的检索中将他们删除释放.

###**Fault Tolerance and Diagnosis -- 容错与诊断**
由于GFS集群是基于大量的普通机器, 数据损坏, 硬盘错误, 宕机等问题必然经常会发生. 为了提高整个文件系统的容错性, master和文件服务器都设计成无论是正常还是异常关停, 都可以在秒级内存储或者恢复当前的数据状态(Fast Recovery). 文件数据的跨集群冗余保证了一个集群整体异常时数据的可靠性. Master的备份由shadow master来完成, 它与master一样以同样的逻辑和顺处理一切请求, 并在master异常的时候成为下一个master. 数据损坏通常通过文件块的checksum来检查.<br/><br/><br/>

####以上是GFS的一些主要思想, 忽略了不少细节, 其中不乏很精妙的设计, 比如错误处理啊, 数据流优化啊等等. 推荐大家有空也去读读原文啦.





[gfs_arc]: /assets/images/gfs_arc.jpg "GFS Architecture"
[gfs_wri]: /assets/images/gfs_wri.jpg "GFS Data Flow"
[gfs]: http://www.eecg.toronto.edu/~ashvin/courses/ece1746/2003/reading/ghemawat-sosp03.pdf