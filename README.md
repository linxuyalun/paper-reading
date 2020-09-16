# paper-reading
## Scheduler

**Medea: Scheduling of Long Running Applications in Shared Production Clusters**

* Issue

  The rise in popularity of **machine learning, streaming, and latency- sensitive online applications** in **shared production clusters** has raised new challenges for cluster schedulers. To optimize their performance and resilience, **these applications require precise control of their placements, by means of complex constraints**, e.g., to collocate or separate their long-running containers <u>across groups of nodes</u>. 

  Since cluster managers are application-agnostic, they have enabled cluster operators to consolidate diverse workloads onto shared clusters.

  In the presence of these applications, the cluster scheduler must attain global optimization objectives, **such as maximizing the number of deployed applications or minimizing the violated constraints and the resource fragmentation, but without affecting the scheduling latency of short-running containers**.

* Solution

  * Our experiments with various LRAs (LONG RUNING APPLICATION e.g., HBase, TensorFlow, and Storm; see §2) reveal that powerful constraints are required to capture interactions between containers and unlock the full potential of LRAs.
  * Due to their long lifetimes, LRAs can tolerate longer scheduling latencies than tra- ditional batch jobs. The second requirement for LRA placement is therefore to allow cluster operators to optimize for global clus- ter objectives, but without impacting the scheduling latency of short-lived containers.
  * A new cluster scheduler designed for the **placement of long- and short-running containers**. 
  * Medea introduces **powerful placement constraints** with formal semantics to capture interactions among containers within and across applications.
  * It follows a novel two-scheduler design: (i) for long-running containers, it applies an **optimization-based approach that accounts for constraints and global objectives**; (ii) for short-running containers, it uses **a traditional task-based scheduler for low placement latency**. 
  * **Expressive, high-level constraints**. Medea enables application owners and cluster operators to specify powerful placement constraints across LRA containers with formal semantics. **Relying on the notions of container tags and node groups**, Medea supports both intra- and inter-application constraints, without requiring knowledge of the cluster’s configuration or of already- deployed LRAs
  * Evaluated on a 400-node cluster, our implementation of Medea on Apache Hadoop YARN achieves placement of long-running applications with significant performance and resilience benefits compared to state-of-the-art schedulers.
  * shared production cluster ？？

**Neptune: Scheduling Suspendable Tasks for Unified Stream/Batch Applications**

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

