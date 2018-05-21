---
title: 分布式运行环境
nav-pos: 2
nav-title: 分布式运行
nav-parent_id: concepts
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

* This will be replaced by the TOC
{:toc}

## Tasks 和 Operator Chains

在分布式执行环境下，Flink 将多个 operator 的 subtask *链化(chain)* 成为 *task*。每个 task 由一个线程执行。
将 operator 链化合并成 task 是一个很有用的优化：它减少了线程间切换和缓冲的开销，提高了整体吞吐量，减少时延。链化操作是可配置的；详细信息参考[链化文档 (chaining docs)](../dev/stream/)。

下图中的数据流例程由五个子任务执行，因此具有五个并行线程。

<img src="../fig/tasks_chains.svg" alt="Operator chaining into Tasks" class="offset" width="80%" />

{% top %}

## 工作管理器(Job Managers)，任务管理器(Task Managers)，客户端(Clients)

Flink运行时包含两类进程:

  - **JobManagers** (也被称作 *masters*) 协同分布式执行节点。分发 task，协同
    checkpoints，协调故障恢复等。

    系统中至少有一个JobManager。 采用高可用性配置可以拥有多个 JobManager，其中一个是 *leader*，其他处于 *standby* 状态。

  - **TaskManagers** (也被称作 *workers*) 用于执行数据流的 *task* (更具体一点, 是执行 subtask), 缓冲并改变 *stream*.

    系统中至少有一个 TaskManager.

JobManagers 和 TaskManagers 能够以各种方式启动：直接在机器上启动[独立集群 (standalone cluster)](../ops/)，在容器里启动，或者由资源管理框架管理启动，例如[YARN](../ops) 或者 [Mesos](../ops)。 TaskManagers 连接到 JobManager，
宣布自己可用，并由 JobManager 下发工作。

**client** 不是运行和程序执行的一部分，但是常用于准备数据流并将其发送到 JobManager。
完成这些任务后，client 可以中断连接，或者保持连接用于接收运行结果。client 作为触发 Java/Scala 程序的一部分运行，或者在命令行进程中执行 `./bin/flink` 运行...

<img src="../fig/processes.svg" alt="The processes involved in executing a Flink dataflow" class="offset" width="80%" />

{% top %}

## Task Slots 和 运行资源

每个 worker (TaskManager) 是一个 *JVM 进程*，并在单独的线程中执行一个或多个 subtask。
为了控制worker能接受多少 task，每个 worker 有所谓的 **task slots** （至少一个）。

每个 *task slot* 代表 TaskManager 的一个固定大小的资源子集。例如，一个拥有3个slot的 TaskManager，会将其管理的内存平均分成三分分给各个 slot。
将资源 slot 化意味着来自不同 job 的 task 不会为了内存而竞争，而是每个 task 都拥有一定数量的内存储备。需要注意的是，这里不会涉及到 CPU 的隔离，slot 目前仅仅用来隔离 task 的内存。

通过调整 task slot 的数量，用户可以定义 task 之间是如何相互隔离的。
每个 TaskManager 有一个 slot ，也就意味着每个 task 运行在独立的 JVM 中 （例如可以在一个独立的容器内启动）。
每个 TaskManager 有多个 slot 的话，也就是说多个 subtask 运行在同一个JVM中。
而在同一个JVM进程中的 task，可以共享 TCP 连接（基于多路复用）和心跳消息，可以减少数据的网络传输。也能共享一些数据结构，一定程度上减少了每个 task 的消耗。

<img src="../fig/tasks_slots.svg" alt="A TaskManager with Task Slots and Tasks" class="offset" width="80%" />

默认情况下，Flink 允许多个 subtask 共享 slot，甚至只要这些 subtask 来自同一个 job，他们可以是不同 task 的 subtask。
结果可能一个 slot 持有该 job 的整个 pipeline。允许 *slot sharing* 有以下两点好处：

  - Flink 集群所需的 task slots 数与 job 中最高的并行度一致。也就是说我们不需要再去计算一个程序总共会起多少个 task 了

  - 更容易获得更加充分的资源利用。如果没有slot共享，那么非密集型操作 *source/map()* 就会占用密集型操作 *window* 同样的资源。
    如果有 slot 共享，在我们的例子中将基础的2个并行度增加到6个，能充分利用 slot 资源，同时保证每个 TaskManager 能平均分配到任务重的 subtask。

<img src="../fig/slot_sharing.svg" alt="TaskManagers with shared Task Slots" class="offset" width="80%" />

APIs 还包括 [*resource group*](../dev/stream) 机制，可用于防止不需要的 slot 共享。

根据经验，默认情况下，比较好的 task slot 数量等于CPU核数。使用超线程技术时，每个 slot 需要2个或更多的硬件线程上下文环境。

{% top %}

## 状态后端

存储在 key/value 索引上的数据结构取决于所选择的 [**状态后端 (state backend)**](../ops/state_backends.md)。一部分状态后端将数据存在
内存中的哈希映射，另一些状态后端用 [RocksDB](http://rocksdb.org) 作为 key/value 存储。
除了定义保存状态的数据结构之外，状态后端还逻辑实现了获取 key/value 状态的时间点快照，并将该快照存储为 checkpoint 的一部分。

<img src="../fig/checkpoints.svg" alt="checkpoints and snapshots" class="offset" width="60%" />

{% top %}

## 保存点 (Savepoints)

用Data Stream API 编写的程序可以从 **保存点(Savepoints)** 恢复执行。savepoint 允许在不会丢失任何状态情况下，更新程序和 Flink 集群。 

[Savepoint](../ops) 是 **手动触发的 checkpoint**, 它获取程序快照并写入状态后端。他们依赖常规的 checkpoint 机制。在执行期间，程序会定期在 worker 节点上生成快照并产生 checkpoints。 
系统恢复时，仅需要最后一个完成的 checkpoint，并且一旦新的 checkpoint 完成，可以安全地丢弃旧的 checkpoint。

savepoint 类似于周期的 checkpoint，除了它们 **由用户触发** 而且在新 checkpoint 完成时 **不会自动失效**。Savepoint 可以由[命令行](../ops)创建，也可以通过[RST API](../monitoring)在取消任务时创建。

{% top %}
