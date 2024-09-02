---
title: "Milvus数据库"
date: 2024-09-02T14:15:48+08:00
lastmod: 2024-09-02T14:15:48+08:00
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

1. 硬件感知优化： 为了提高milvus在不同环境中的性能，Milvus专门优化了在许多硬件架构和平台上的性能表现，包括AVX512、SIMD、GPUs以及NVMe SSD。
2. 高级搜索算法： Milvus支持广泛的in-memory和on-disk索引和搜索算法，包括IVF、HNSW、DiskANN等，所有这些算法都被深度优化过。拿FAISS和HNSWLib来说，Milvus比一般实现要更快30% - 70%。
3. C++搜索引擎： 80%向量数据库的性能决定于其搜索引擎。Milvus对关键组件使用C++进行开发，因其高性能、底层优化以及高效的资源管理。最重要的是，Milvus组合使用了大量的硬件感知优化代码，从汇编层面的向量话到多线程并发和调度，尽力榨干硬件的性能潜力。
4. 面向列（Column-Oriented）存储：Milvus是一个Column-Oriented的向量数据库。当执行一个查询操作时，面向列的数据库只读取特定的字段，而不需要读取一整个行数据，从而大大的减少了无效数据的访问。还有就是，以列为基础的操作很容易向量化，允许一次奖操作应用到整个列上，更进一步提高了性能。

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
