[TOC]
#GFS
##1.Introduction
    ①组件错误是平常的不是异常。
        错误容忍、恢复
    ②文件很大。
        块的大小、I/O性能(适合大文件、高带宽、不适合小文件与低延迟)
    ③一次写入，多次读取。
        一次写入(append)，反复读取
    ④应用与文件系统API协同设计。
        多client的写操作的原子性(队列、多路合并)

##2.Overview
###2.1 Architecture
    架构：一个master，多个chunkserver。
    ①master维护所有文件的元数据（块的命名空间、访问控制、文件到块的映射、块的当前位置、块的管理、失效chunk垃圾回收、块在chunkserver间的移动） + 心跳机制。
    ②文件被分成固定大小的chunk块存储在chunkservers中。
    ③client读写与master交流元数据，再与chunkserver直接交互。
    ④client(缓存文件元数据)与chunkserver都不缓存文件数据，极小收益。

###2.2 Single Master
    简化设计、方便master统筹全局信息以做出块的安置和块副本的决策。
    最小化读写操作与master的交互，client只与master交互元数据信息并缓存：client请求文件名和chunk索引（=文件名+字节偏移），master返回chunk handle和副本位置。client与最近的chunkserver交互。

###2.3 Chunk Size
    64MB/块，比传统的文件系统的块大得多的多。
    利：
        减少client与master的交互次数；
        因操作多，故可用TCP持续性连接来降低网络开销；
        减少了master需要存储的元数据；
    弊：
        小文件包含少量chunk(甚至一个)，会成为hotspot，没有理解这一段[*]

###2.4 Metadata
    文件和chunk的命名空间、文件到chunk的映射、每个chunk的副本的位置。

####2.4.1 In-Memory Data Structure    
    三者都存储在master内存中（快速，增加master的内存还是划算的），并且前两者持久化到操作日志中，操作日志也遵循多副本机制。

####2.4.2 Chunk Locations
     master不持久存储chunkserver的副本信息，而是间歇性同步更新。

####2.4.3 Operation Log
    记录metadata的修改记录；
    定义并发操作的逻辑时间线；
    master本地和remote机器的操作日志都写完后，再响应client的操作。
    通过日志回放恢复文件系统，日志文件增长到一定大小后，master设置一个检查点。设置检查点时master切换到一个新的log中（包含所有旧的操作）。恢复数据时，只需要找到最后一个完成的checkpoint和其之后的log files。

###2.5 Consistency Model
    namaspace锁定确保原子性和正确定；
    operation log定义了一个全局的操作顺序；