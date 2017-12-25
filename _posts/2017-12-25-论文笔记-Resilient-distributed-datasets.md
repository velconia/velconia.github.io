---
layout: post
title: 论文笔记: Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing
date: 2017-12-25 19:34:34
categories: 技术调研
tags: 大数据, Spark
---
# Spark RDD 论文解读

## Paper Link
[Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing](https://cs.stanford.edu/~matei/papers/2012/nsdi_spark.pdf)

## 背景:
1. 数据增长 —> 单机处理能力没跟上 —> 多机, 集群处理;
2. 多机处理的编程方法和原来不一样, 需要新的编程框架 —> Hadoop, F1, MillWheel, Storm, Impala, Pregel, BSP

## 现有问题:
以上系统都是专用系统, 只解决专用问题, 但还遗留了很多问题未解决:
1. 分布式处理的部分, 很多都是重复开发; (Spark提供一个通用框架, 分布式都由Spark处理, 不用专用系统考虑, 省去重复开发)
2. 解决大问题时, 需要多个系统合作, 但合作的代价是数据导入导出, 很昂贵; (Spark 提供一个分布式的通用数据集合表示, 省去数据在多系统的导入导出, 并且很高效)
3. 这些系统不解决已有代码的适配问题; (说实话, Spark也没解决这个问题)
4. 多个系统合作时, 集成环境部署, 管理, 也是需要额外开发的; (Spark就搭一个, 当然简单)

## RDD
#### RDD设计: 
1. RDD只能通过数据或者别的RDD构建, 并且构建之后是不可变的;
2. RDD包含以下几个层面: 
    1. transformation: e.g.. map, filter
    2. action: e.g.. collect
    3. persistence: memory or disk or any other storage
    4. partitioning: how to partition data in RDD
3. 只支持粗粒度变换, 不支持细粒度的更改某个条目;

#### RDD优点(和 Distributed Shared Memory 比较):
1. 只支持粗粒度变换, 降低了Fault Tolerence的难度, 可以和 MapReduce 一样重跑 Straggler, 并且是针对 Partition 的 Straggler;
2. 限制了用户的使用, 快速的细粒度操作是不能使用 RDD 来实现的;

所以RDD针对的是大量数据计算的场景;

#### RDD的表示:
DAG图是RDD的表示方法, RDD内部包含以下最基本的接口, 用以实现新的transformation方法:
1. partitions: RDD的原子部分;
2. dependencies: RDD的parents RDD;
3. preferredLocation(p)
4. iterator(p, parentIters)
5. partitioner
map, union, HDFS files 等方法, 都是基于实现或使用这些接口来实现的;

## Spark的实现:
#### Scheduler:
类似于 [Dryad](http://budiu.info/work/eurosys07.pdf) 的实现;
Spark把RDD之间的依赖分为 Narrow Dependencies(map, union..) 和 Wide Dependencies(reduce), 原因如下:
1. narrow dep都是partion之间一对一的映射, 而 wide dep都是多对多的, 这导致 narrow dep之间可以随便串联, 并且更容易实现并发执行, 只要上游partition OK了下游partition就可以继续计算, 而不需要等待 RDD全部OK;
2. narrow dep的错误恢复也只需要恢复依赖的上游部分, 而不需要像 wide dep一样恢复全部;
RDD的DAG图会被分为多个 Stage, Stage内部是尽量多的 narrow dep 连接起的 RDD, 碰到 wide dep就变成下一个Stage, 这样Stage之间就不需要互相等待, 可以并发执行了;

#### Interpreter:
Scala 支持交互式的 Interpreter, 交互式的终端会修改用户的 Scala 代码, 每一行都编译成一个类, 扔到 JVM 里去执行;
Spark修改了 Scala interpreter 的实现: 
1. 提供了 HTTP 接口, 让跨 Driver 与 Worker 之间执行变为可能;
2. 优化了Scala的逻辑, 不要每行都编译成一个类, 必要的编译成一个类就行;

#### Memory Management:
RDD有三种persistence方式:
1. in-memory java obj: JVM直接调用, 速度快
2. in-memory serialized data: 不能直接调用, 但是占内存小
3. on-disk storage

新的RDD的新partition如果需要的存储不够, 那么就会用 LRU 算法回收之前的 partition;

Tachyon用于RDD之间共享, 提高Spark集群的内存利用率;

#### Checkpoint:
原因:
对于一些长lineage(血缘)的DAG图来说, 重新计算RDD还是太费时, 所以RDD会做 checkpoint;
优势:
RDD是不变的, 所以不需要快照RDD, 直接在后台慢慢做checkpoint就行;
Todo:
自动化checkpoint RDD, 系统统计出 RDD 的计算时间, 就应该知道需要对那些 RDD 做checkpoint;

