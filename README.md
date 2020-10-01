# paper-reading
## Scheduler

**[Medea: Scheduling of Long Running Applications in Shared Production Clusters](scheduler/medea/medea.md)**

* Issue

  The rise in popularity of **machine learning, streaming, and latency- sensitive online applications** in **shared production clusters** has raised new challenges for cluster schedulers. To optimize their performance and resilience, **these applications require precise control of their placements, by means of complex constraints**, e.g., to collocate or separate their long-running containers <u>across groups of nodes</u>. 

  Since cluster managers are application-agnostic, they have enabled cluster operators to consolidate diverse workloads onto shared clusters.

  In the presence of these applications, the cluster scheduler must attain global optimization objectives, **such as maximizing the number of deployed applications or minimizing the violated constraints and the resource fragmentation, but without affecting the scheduling latency of short-running containers**.

  At the same time, **placing LRAs, along with batch jobs, in shared clusters is appealing to reduce cluster operational costs, avoid unnecessary data movement, and enable pipelines involving both classes of applications**.

* Solution

  * Our experiments with various LRAs (LONG RUNING APPLICATION e.g., HBase, TensorFlow, and Storm; see §2) reveal that powerful constraints are required to capture interactions between containers and unlock the full potential of LRAs.
  * Due to their long lifetimes, LRAs can tolerate longer scheduling latencies than tra- ditional batch jobs. The second requirement for LRA placement is therefore to allow cluster operators to optimize for global clus- ter objectives, but without impacting the scheduling latency of short-lived containers.
  * A new cluster scheduler designed for the **placement of long- and short-running containers**. 
  * Medea introduces **powerful placement constraints** with formal semantics to capture interactions among containers within and across applications.
  * It follows a novel two-scheduler design: (i) for long-running containers, it applies an **optimization-based approach that accounts for constraints and global objectives**; (ii) for short-running containers, it uses **a traditional task-based scheduler for low placement latency**. 
  * **Expressive, high-level constraints**. Medea enables application owners and cluster operators to specify powerful placement constraints across LRA containers with formal semantics. **Relying on the notions of container tags and node groups**, Medea supports both intra- and inter-application constraints, without requiring knowledge of the cluster’s configuration or of already- deployed LRAs
  * Evaluated on a 400-node cluster, our implementation of Medea on Apache Hadoop YARN achieves placement of long-running applications with significant performance and resilience benefits compared to state-of-the-art schedulers.
  * shared production cluster ？？

[**Neptune: Scheduling Suspendable Tasks for Unified Stream/Batch Applications**](scheduler/neptune/neptune.md)

* Issue
  * A recent trend is to unify different computation types as part of a single stream/batch application that combines latency-sensitive (“stream”) and latency-tolerant (“batch”) jobs.
  * Existing execution engines, however, were not designed for unified stream/batch applications. As we show, they fail to schedule and ex- ecute them efficiently while respecting their diverse requirements.
* Solution
  * The stream jobs of these applications must be executed with minimum delay to achieve low latency, which means that batch jobs may have to be preempted.
  * Given that batch and stream jobs share the same runtime, our key idea is to **employ application-specific mechanisms and policies that dynamically prioritize stream jobs without <u>unduly penalizing</u>(过度的处罚) batch jobs**. 
  * Neptune, an execution framework for stream/batch applications that dynamically prioritizes tasks to achieve low la- tency for stream jobs. 
  * Neptune employs coroutines as a l**ightweight mechanism for suspending tasks without losing task progress**. It couples this fine-grained control over CPU resources with a **locality-and memory-aware** (LMA) scheduling policy to determine which tasks to suspend and when, thereby sharing executors among heterogeneous jobs.
  * **Lightweight suspendable tasks**. To prioritize latency-sensitive tasks, Neptune suspends tasks that belong to batch jobs on-demand. As a lightweight task preemption mechanism, **Neptune uses coroutines, which avoid the overhead of thread synchronization**. **Coroutines can suspend batch tasks within milliseconds, thus reducing head-of-line blocking for latency-sensitive tasks. When tasks are resumed, they preserve their execution progress**. To the best of our knowledge, Neptune is the **first distributed dataflow system** to use **coroutines** for task implementation and scheduling.

**Pigeon: an Effective Distributed, Hierarchical Datacenter Job Scheduler**

* Issue

  In today’s datacenters, job heterogeneity makes it difficult for schedulers to **simultaneously meet latency requirements and maintain high resource utilization**. 

  The key issues are the **scalability in centralized schedulers, ineffective and inefficient probing and resource sharing in both distributed and hybrid schedulers**.

  It is common practice to collocate short and long jobs in datacenter management, but meeting the diverse needs of heterogeneous jobs remains a critical challeng

* Solution

  * Pigeon, a distributed, hierarchical job scheduler based on a two-layer design.
  * Pigeon divides workers into groups, each managed by a **separate** master. In Pigeon, upon a job arrival, a distributed scheduler directly distribute tasks evenly among masters with minimum job processing overhead, hence, preserving highest possible scalability. Meanwhile, each master manages and distributes all the received tasks centrally, oblivious of the job context, allowing for full sharing of the worker pool at the group level to maximize multiplexing gain. 

* 这篇文章在一开始谈论 short 和 long job 各自的特点的时候描述的非常清楚，可以作为有效的参考。同样的，它在 introduction 里面就已经充分介绍了自己的这个方案是怎么操作的。

**THEMIS: Fair and Efficient GPU Cluster Scheduling**

* Issue

  Training individual ML models is time- and resource-intensive with each training job typically executing in parallel on a number of GPUs.

  Significant contention ensues when multiple such workloads are run atop a shared cluster of GPUs. A key ques- tion is how to fairly apportion GPUs across workloads. We find that established cluster scheduling disciplines are a poor fit because of ML workloads’ unique attributes: ML jobs have long-running tasks that need to be gang-scheduled, and their performance is sensitive to tasks’ relative placement.

  However, today, there are no ML workload-specific mechanisms to share a GPU cluster in a *fair* manner.

  If there are a total *N* users sharing a cluster *C*, every user’s performance should be no worse than using a private cluster of size *C*/N（一个多租户不公平的问题）。

  Quincy [18], DRF [8], and Carbyne [11]. However, these techniques were designed for big data work- loads, and while they are used widely to manage GPU clusters today, they are far from effective.

* Solution

  * THEMIS, a new scheduling framework for ML training workloads. It’s **GPU allocation policy** enforces that ML workloads complete in *a finish-time fair* manner, a new notion we introduce. 
  * To capture placement sensitivity and ensure efficiency, THEMIS uses a **two-level scheduling archi- tecture** where ML workloads bid on available resources that are offered in an *auction* run by a central arbiter. Our auction design allocates GPUs to winning bids by trading off fairness for efficiency in the short term, but ensuring finish-time fair- ness in the long term. 

**Autopilot: workload autoscaling at Google**

* 要解决的问题：
* 程序在集群中跑的时候，一个超过给定资源限制的任务可能会被截流或者被干掉，导致终端用户的延迟，所以一般来说，需要有集群管理者，主动分配一些多一点的资源，避免资源不够。

## Serverless

**Cloud Programming Simplified: A Berkeley View on Serverless Computing**

一篇关于 serverless 的综述，本文见[[译]简化云编程：伯克利关于Serverless计算的观点](https://zhuanlan.zhihu.com/p/76180907)。

**[SOCK: Rapid Task Provisioning with Serverless-Optimized Containers](serverless/sock/sock.md)**

sock 这篇文章主要做的贡献是针对 container 冷启动过长的问题进行的一系列优化，首先它认为 serverless 条件下现有的 docker 实现很多属性不需要了，具体的，

* 使用 bind 来去掉 aufs
* 使用了 chroot 取代了 mount namespace
* 去掉了 net ns，因为可以依靠进程间通信
* reuse cgroup

当然，这篇文章还做了其他优化，但是这部分优化对我来说是相对帮助最大的部分。

## Streaming Processing

[**Books: Streaming System**](computing/streaming-system/streaming-system.md)

一本关于流处理大家非常推荐的书籍。

[**The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive Scale, Unbounded, Out of Order Data Processing**](computing/dataflowmodel/dataflowmodel.md)

* 要解决的问题：
  * 实践表明，我们**永远无法同时优化数据处理的准确性、延迟程度和处理成本等各个维度**。因此，数据工作者面临如何协调这些几乎相互冲突的数据处理技术指标的窘境，设计出来各种纷繁的数据处理系统和实践方法。