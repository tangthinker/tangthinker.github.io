---
title: "Milvus数据库"
date: 2024-08-09T14:15:48+08:00
lastmod: 2024-08-09T14:15:48+08:00
draft: false
keywords: ["数据库", "Milvus"]
description: "Milvus数据库介绍"
tags: []

categories: ["random"]
author: "Tangthinker"

# Uncomment to pin article to front page
# weight: 1
# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: false
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false

# Uncomment to add to the homepage's dropdown menu; weight = order of article
# menu:
#   main:
#     parent: "docs"
#     weight: 1
---

<!--more-->


# 简介

## 什么是Milvus？

Milvus是一个高性能、高扩展性向量数据库，并且可以运行在任何规模的机器上。

Milvus是一个开源项目，使用Apache 2.0许可证。

## Milvus为什么这么快？

1. **硬件感知优化**： 为了提高milvus在不同环境中的性能，Milvus专门优化了在许多硬件架构和平台上的性能表现，包括AVX512、SIMD、GPUs以及NVMe SSD。
2. **高级搜索算法**： Milvus支持广泛的in-memory和on-disk索引和搜索算法，包括IVF、HNSW、DiskANN等，所有这些算法都被深度优化过。拿FAISS和HNSWLib来说，Milvus比一般实现要更快30% - 70%。
3. **C++搜索引擎**： 80%向量数据库的性能决定于其搜索引擎。Milvus对关键组件使用C++进行开发，因其高性能、底层优化以及高效的资源管理。最重要的是，Milvus组合使用了大量的硬件感知优化代码，从汇编层面的向量话到多线程并发和调度，尽力榨干硬件的性能潜力。
4. **面向列（Column-Oriented）存储**：Milvus是一个Column-Oriented的向量数据库。当执行一个查询操作时，面向列的数据库只读取特定的字段，而不需要读取一整个行数据，从而大大的减少了无效数据的访问。还有就是，以列为基础的操作很容易向量化，允许一次奖操作应用到整个列上，更进一步提高了性能。

## Milvus架构

2022年，Milvus支持十亿尺度的向量，在2023年，Milvus支持千亿界别的向量并提高持续的稳定性。为300多家主要的企业提供支持，包括Paypal、Shopee、Airbnb、eBay、NVIDIA等。

Milvus的云原生和高解藕系统架构保证了整个系统可以随着数据增长而持续扩张。

![Milvus架构图](/img/random/milvus-architecture.png)

Milvus本身是完全无状态的声音可以很容易的借助Kubernetes或者公有云来进行伸缩。还有就是，Milvus的组件是解藕良好的，有三个主要的任务：搜索（search）、数据插入（data insertion）、索引/压缩（indexing/compaction），由于其逻辑上分离，这三个任务很容易的被设计成并行处理。这也保证了相关的查询节点、数据的以及索引节点可以独立的进行伸缩、性能优化以及成本优化。

## Milvus支持的搜索类型

1. ANN Search： 找到Top K个与查询向量接近的向量。
2. Filtering Search： 在特定的过滤条件下执行ANN Search。
3. Range Search： 在查询向量的特定范围内进行向量查找。
4. Hybrid Search： 基于多个向量字段执行ANN Search。
5. Keyword Search： 基于BM25进行关键字搜索。
6. Reranking： 基于额外的标准或算法对搜索结果进行重排序，改善最初的ANN Search结果。
7. Fetch： 基于主键取回数据。
8. Query： 使用特定搜索表达式取回数据。

解释：

ANN Search： 即Approximate Nearest Neighbor Search，是一种基于向量空间的搜索算法，通过计算向量之间的距离来找到最相似的向量。Milvus支持基于向量的ANN Search，包括IVF、HNSW、PQ、OPQ等。
与精确的NN算法即Nearest Neighbor Search相比，ANN允许在寻找最近邻是稍微牺牲精度以换取更高的计算效率。这对于处理大规模数据特别重要，因为高位空间中精确搜索的计算成本非常高。

Milvus支持ANN算法包括：

1. IVF（Inverted File Index）： 将数据集划分为多个簇，查询时仅在相关的簇中查找，提升搜索效率。
2. HNSW（Hierarchical Navigable Small World Graph）：一种基于图的算法，适用于高维数据的快速相似性搜索。
3. Flat： 也称为Brute-Force Search， 即遍历整个向量空间，找到与查询向量距离最近的向量。可提供小规模数据集上的精确的最近邻搜索。
4. ANNOY（Approximate Nearest Neighbors Oh Yeah）： 一种基于树的算法，适用于高维数据的快速相似性搜索。
5. NSG（Navigating Spreading-out Graph）：通过优化图的接哦股，实现高效的搜索和插入操作。

# 架构

## 整体架构图

![Milvus架构图](/img/random/milvus-architecture-1.png)

## 主要组件

### 单机模式

![Milvus单机架构图](/img/random/standalone-architecture.png)

Milvus单机模式包括三个组件：
1. Milvus：核心功能模块
2. Mate Sotre： 元数据引擎，存储和访问Milvus内部组件包括代理、索引节点等等元数据。
3. Object Storage：存储引擎，负责Milvus等数据持久化。

上图中：

Cooridinator组件：

1. Root Coordinator(root coord)：处理Data definition language(DDL)和Data Control language(DCL)的请求，比如创建或删除Collection、Partition、Index等。
2. Query Coordinator(query coord)：为查询节点提供管理拓扑关系和负载均衡的能力。
3. Data Coordinator(data coord)：管理数据节点和索引节点的拓扑学关系，维护元数据，触发数据刷新、压缩和索引重建以及其他的后台数据操作。

Worker Nodes组件：

工作者节点是无声的执行者，默默执行来自协作者服务（coordinator）的指令和来自代理（proxy）的Data Manipulation language（DML）命令。

工作者节点得益于其存算分离的设计，是无状态且可以很容易在k8s上的进行扩容、故障恢复。

工作者节点包括三种类型：

1. Query Node： 查询节点通过订阅日志broker取回增长式（incremental）的日志数据，并将其转化为增长式片段（growing segments）。
2. Data Node: 通过订阅日志broker取回增长式的日志数据，处理不同的请求并且打包日志数据为日志快照并将其存储在对象存储中。
3. Index Node： 索引节点构建索引。索引节点不需要内存驻留，并可以通过无服务框架来实现。

Storage：

Storage是一整套系统，负责数据持久化。包括元数据存储，日志代理（log broker）和对象存储。

1. Mate Storage： 元数据存储保存了一些数据结构的元数据快照，包括collection、schema和消息消费检查点。**元数据的存储需要非常高的可用性和强一致性并且要支持事务**，所以Milvus使用etcd来存储元数据。Milvus也使用etcd来进行服务的注册和检查。
2. Object Storage： 对象存储保存了一些文件快照，包括日志文件、向量数据或常规数据的索引文件和间接查询数据文件等。Milvus目前使用MinIO作为对象存储。然而，对象存储在数据访问上有很高的延迟和性能损耗，为了提高其性能减少成本，Milvus计划基于内存或SSD的缓冲池来实现冷热数据的分离。
3. Log Broker： 日志代理是一个支持重放（palyback）的发布者订阅者（pub-sub）系统。负责数据持久化传输和事件提醒。同时保证当工作者节点从系统宕机中恢复时的增长式数据的完整性。Milvus集群使用Pulsar作为Log Broker。Milvus单机模式使用RocksDB作为Log Broker。此外，日志代理可以很容易的使用其他流式数据处理平台替代，比如说Kafka。


### 集群架构

![Milvus集群架构图](/img/random/cluster-architecture.png)

Milvus Cluster包含7个微服务组件和3个三方依赖。所有服务可以在k8s上进行单独的部署。

微服务组件包括：
1. Root Coordinator
2. Query Cooordinator
3. Data Coordinator
4. Query Node
5. Data Node
6. Index Node
7. Proxy

三方依赖：

1. Meta Store： 存储集群中各种组件的元数据，例如etcd。
2. Object Storage： 负责集群中大文件的数据持久化，比如索引和二进制日志文件。使用S3/MinIO。
3. Log Broker： 管理最近修改数据操作的日志，输出流式日志，并且提供日志发布者-订阅者服务。使用Pulsar。

### 数据处理流程

这部分包含Milvus中的数据插入、索引构建和数据查询。

#### 数据插入流程

在Milvus中可以为每个collection设置分片（shard）数量，每个分片对应一个虚拟通道（vchannel）。

如下图所示，Milvus会为每一个vchannel在log broker中分配一个物理通道（pchannel）。

所有insert/delete请求都会通过主键hash的方式路由到不同的分片。

![Milvus VChannel](/img/random/milvus-channel.png)

DML请求校验被移动到proxy中进行处理，因为Milvus并没有复杂的事务。

![Milvus Write Log](/img/random/milvus-write-log.png)

#### 索引构建流程

![Milvus Index Build](/img/random/milvus-index-build.png)

索引构建的过程在索引节点上执行。为了避免因为数据更新的导致的频繁的索引构建，一个collection在Milvus中被进一步分割成了segments，每一个segment对应一个索引文件。

Milvus支持为每个向量字段、常规字段以及主键字段构建索引。

索引构建的输入和输出都是使用对象存储进行：索引节点从对象存储中加载一个segment的日志快照（log snapshots）到内存中，反序列化相应的数据和元数据进行索引构建，当索引构建完成时序列化索引数据并将其写入对象存储中。

