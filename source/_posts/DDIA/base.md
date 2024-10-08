---
title: DDIA-数据应用系统
data: 2024-07-26
cover: /img/blog/blog12.jpg
categories:
- 大数据
tags:
- DDIA
- big data
---

## 背景

当今许多新型应用都属于数据密集型 (data-intensive) ，而不是计算密集型(compute-intensive) 。对于这些类型应用，CPU的处理能力往往不是第一限制性因素，关键在于数据量、数据的复杂度及数据的快速多变性。

<!--more-->

数据密集型应用通常也是基于标准模块构建而成，每个模块负责单一的常用功能。例如，许多应用系统都包含以下模块：
- 数据库: 用以存储数据，这样之后应用可以再次访问。
- 高速缓存: 缓存那些复杂或操作代价昂贵的结果，以加快下一次访问。
- 索引: 用户可以按关键字搜索数据并支持各种过着。
- 流式处理: 持续发送消息至另一个进程，处理采用异步方式。
- 批处理: 定期处理大量的累积数据。

越来越多的应用系统需求广泛，单个组件往往无法满足所有数据处理与存储需求。因而需要将任务分解，每个组件负责高效完成其中一部分，多个组件依靠应用层代码驱动有机衔接起来。

![](../../img/blogs/DDIA/一/1png.png)

## 设计原则

设计数据系统或数据服务时，一定会碰到很多环手的问题。例如，当统内出现了局部失效时，如何确保数据的正确性与完整性?当发生系统降级 (degrade) 时，该如何为客户提供一致的良好表现?负载增加时，系统如何扩展? 友好的服务API该如何设计?

下面是大多数软件系统都极为重要的三个问题

### 可靠性

#### 定义

可靠性指的是系统在各种条件下都能正确执行功能的能力。这包括有效处理硬件故障、软件错误和人为操作失误。

确保系统即便在部分组件失效的情况下，也能持续提供服务，从而避免服务中断带来的损失。

#### 适用场景
适用于对业务连续性要求极高的场景，如金融服务、在线交易平台等。

采用冗余系统和自动故障转移技术，即使部分服务器故障，系统也能自动切换到备用服务器继续运行，如多数大型云服务提供商所采用的策略。

#### 优劣
相比于传统的单一服务器系统，采用可靠性设计的分布式系统提供了更高的故障容忍度和业务连续性，但可能会增加系统复杂性和维护成本。

### 可扩展性

#### 定义

可扩展性是指系统在负载增加时，能够通过增加资源（横向扩展或纵向扩展）来维持性能水平的能力。

应对用户增长或数据量增加带来的性能挑战，确保系统性能不会因扩展而降低。

#### 适用场景
适用于用户基数和数据快速增长的应用，如社交媒体平台、大数据分析等。

在分布式数据库中，通过增加更多的服务器节点来分担查询压力，例如Apache Cassandra数据库就是基于这种模型设计的。

#### 优劣
与单节点扩展相比，横向扩展能够提供更高的灵活性和可靠性，但管理上更为复杂，需要更多的网络和同步机制。

### 可维护性

#### 定义

可维护性涉及到使系统易于操作并保持运行，以及便于开发者更新功能和修复错误。

简化系统管理工作，减少维护成本和时间，快速响应市场变化。

#### 适用场景
适用于需要频繁更新和维护的系统，如持续集成的软件开发环境。

采用微服务架构，将服务解耦，每个服务独立运行，独立维护，从而使整体系统更易于管理和扩展。

#### 劣势
相比于传统的单体架构，微服务架构提高了系统的灵活性和可维护性，但也可能带来服务间通信和数据一致性的挑战。