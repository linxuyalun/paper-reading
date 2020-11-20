# paper-reading
## Machine Learning Frameworks

[**Clipper: A Low-Latency Online Prediction Serving System**](ml-framework/clipper/clipper.md)

首先一般来说我们谈到 ML 框架我们想到的都是怎么优化这个训练流程，确实很少去考虑模型的放置，这篇文章告诉我们，模型放置也很重要，这在一定程度上扩展了我们的视野。

Clipper 它干的第一件有意义的事情是它把所有那些混沌复杂的东西简单化了，它用一个中间件去解耦，它抽象了一个统一的模型放置 API，简化了整个部署流程，道出了软件工程的本质是抽象。

CLipper 的另一个亮点是自适应 Batching，但是这个事情它的解决策略非常的简单粗暴，但是它的效果很好，所以有一个完整的系统，方法不需要多复杂，有一个好的实验优化结果，照样上顶会。

但是 Clipper 实际上并不涉及硬件资源的配置，可以预想到的是，不论是什么模型肯定都优先希望自己的模型获得最好的硬件资源嘛，在 Clipper 中这部分内容还是需要用户的主动配置，这也完全可以理解，可以预想到的是，如果模型被放置在配置比较差的机器上，那么它的吞吐量和延迟肯定比较高。不过既然我们可以 AIMD 的方法去做自适应 batching，这同样也可以启发我们用类似的方法自动的去调度一些模型的放置，比如说在运行过程中发现哪些模型是常被选上的，那些模型正确率很高，可以让系统自动的去重新放置这个模型，计算吞吐量和延迟的变化，从而得到一个更高效的放置系统。

[**Ray: A Distributed Framework for Emerging AI Applications**](ml-framework/ray/ray.md)

Ray 突出了两点贡献，首先第一点，它把 task 和 actor 都抽象出来放在了一个统一的系统以同时部署 RL 和 模拟环境。它的整个思想抽象的很好并且 API 设计得也很简单，这是它重要的贡献之一。我们其实就简单两页介绍了一下，主要还是专业不对口。

第二点，它提出了整个 Global Control Store，是其他组件都无状态化了，这种方法其实已经见的蛮多了，它另外一点是一个自底向上的调度器，从而做到尽可能的降低延迟。

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

## Streaming

[**Books: Streaming System**](streaming/streaming-system/streaming-system.md)

一本关于流处理大家非常推荐的书籍。

[**The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive Scale, Unbounded, Out of Order Data Processing**](streaming/dataflowmodel/dataflowmodel.md)

提出了一个抽象的流模型，降维打击。

[**Apache FlinkTM: Stream and Batch Processing in a Single Engine**](streaming/flink/flink.md)

这篇文章主要是对 flink 架构的介绍，其中各种对于流处理中抽象的思考其实基本可以参见 dataflow model 这篇文章。这个架构的模型也比较粗略，其中比较值得关注的就是数据流的流动，已经 back pressure 的应对状况。而另外一个很重要的容错，在下面这篇论文。

**[Lightweight Asynchronous Snapshots for Distributed Dataflows](streaming/flink/abs.md)**

异步全局快照算法，barrier 的思想很有道理，异步的处理也很妙。

**[Turbine: Facebook’s Service Management Platform for Stream Processing](streaming/turbine/turbine.md)**

这篇细节比较丰富，没有什么特别有意思的点，但是很完整。

**[TerseCades: Efficient Data Compression in Stream Processing](streaming/tersecades/tersecades.md)**

18 年的论文，相比之前其他各种的优化，这篇文章是第一篇提出使用压缩算法来加速效率的，整个压缩算法很简单，采用的是 Base + Delta 算法，它的优势体现在快并且可以对压缩数据进行直接处理。但是压缩算法本身并不难，重要的是它的切入点是很好的一个点子。

**[A Survey on the Evolution of Stream Processing Systems](streaming/survey/survey.md)**

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

**[*Le Taureau*: Deconstructing the Serverless Landscape & A Look Forward](serverless/landscape/landscape.md)**

> todo

**[SAND: Towards High-Performance Serverless Computing](serverless/sand/sand.md)**

> Todo

[**An Overview of Anna and Cloudburst**](serverless/rise/rise-view.md)

> RISE 实验室系列文章，RISE Lab [主页](https://hydro-project.github.io/)。

- *[Optimizing Prediction Serving on Low-Latency Serverless Dataflow](https://arxiv.org/pdf/2007.05832.pdf)*. V. Sreekanti, H. Subbaraj, C. Wu, J. E. Gonzalez, J. M. Hellerstein. *arxiv:2007.05832*. 2020.
- *[Cloudburst: Stateful Functions-as-a-Service](http://www.vldb.org/pvldb/vol13/p2438-sreekanti.pdf)*. V. Sreekanti, C. Wu, X. C. Lin, J. Schleier-Smith, J. M. Faleiro, J. E. Gonzalez, J. M. Hellerstein, A. Tumanov. *arxiv:2001.04592*. 2020.
- *[Transactional Causal Consistency for Serverless Computing](https://dl.acm.org/doi/10.1145/3318464.3389710)*. C. Wu, V. Sreekanti, J. M. Hellerstein. *SIGMOD* 2020.
- *[A Fault-Tolerance Shim for Serverless Computing](https://dl.acm.org/doi/abs/10.1145/3342195.3387535)*. V. Sreekanti, C. Wu, S. Chhatrapati, J. E. Gonzalez, J. M. Hellerstein, J. M. Faleiro. *EuroSys* 2020. *To appear.*
- *[Autoscaling Tiered Cloud Storage in Anna](http://www.vldb.org/pvldb/vol12/p624-wu.pdf)*. C. Wu, V. Sreekanti, J.M. Hellerstein. *VLDB* 2019. (Preprint: [arXiv:1809.00089](https://arxiv.org/abs/1809.00089).)
- *[Serverless Computing: One Step Forward, Two Steps Back](http://cidrdb.org/cidr2019/papers/p119-hellerstein-cidr19.pdf)*. J.M. Hellerstein, J. Faleiro, J.E. Gonzalez, J. Schleier-Smith, V. Sreekanti, A. Tumanov, C. Wu. *CIDR* 2019. (Preprint: [arXiv:1812.03651](https://arxiv.org/abs/1812.03651).)
- *[Anna: A KVS for Any Scale](https://ieeexplore.ieee.org/document/8640246)*. C. Wu, J. M. Faleiro, Y. Lin, J. M. Hellerstein. *ICDE* 2018, *TKDE* 2019.

