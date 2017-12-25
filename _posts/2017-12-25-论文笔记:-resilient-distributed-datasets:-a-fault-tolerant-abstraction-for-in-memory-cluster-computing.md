---
layout: post
title: 论文笔记: Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing
categories:
tags:
---
# Spark RDD 论文解读

## paper ref
https://cs.stanford.edu/~matei/papers/2012/nsdi_spark.pdf

## 背景:
1. 数据增长 —> 单机处理能力没跟上 —> 多机, 集群处理;
2. 多机处理的编程方法和原来不一样, 需要新的编程框架 —> Hadoop, F1, MillWheel, Storm, Impala, Pregel, BSP

## 现有问题:
以上系统都是专用系统, 只解决专用问题, 但还遗留了很多问题未解决:
1. 分布式处理的部分, 很多都是重复开发; (Spark提供一个通用框架, 分布式都由Spark处理, 不用专用系统考虑, 省去重复开发)
2. 解决大问题时, 需要多个系统合作, 但合作的代价是数据导入导出, 很昂贵; (Spark 提供一个分布式的通用数据集合表示, 省去数据在多系统的导入导出, 并且很高效)
3. 这些系统不解决已有代码的适配问题; (说实话, Spark也没解决这个问题)
4. 多个系统合作时, 集成环境部署, 管理, 也是需要额外开发的; (Spark就搭一个, 当然简单)

## RDD: 
设计:
1. RDD只能通过数据或者别的RDD构建, 并且构建之后是不可变的;
2. RDD包含以下几个层面: 
    1. transformation: e.g.. map, filter
    2. action: e.g.. collect
    3. persistence: memory or disk or any other storage
    4. partitioning: how to partition data in RDD
3. 只支持粗粒度变换, 不支持细粒度的更改某个条目;
和 Distributed Shared Memory 比较:
1. 只支持粗粒度变换, 降低了Fault Tolerence的难度, 可以和 MapReduce 一样重跑 Straggler, 并且是针对 Partition 的 Straggler;
2. 限制了用户的使用, 快速的细粒度操作是不能使用 RDD 来实现的;

```所以RDD针对的是大量数据计算的场景;```

Spark 架构 (DDL 1.6):

