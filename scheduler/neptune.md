* Issue

  * A recent trend is to unify different computation types as part of a single stream/batch application that combines latency-sensitive (“stream”) and latency-tolerant (“batch”) jobs.

  * Existing execution engines, however, were not designed for unified stream/batch applications. As we show, they fail to schedule and ex- ecute them efficiently while respecting their diverse requirements.

  * 变革：一开始分开的 batch 和 stream 框架，前者侧重高吞吐量，后者侧重低处理延迟，基于这样的限制，由于必须管理多个系统并在它们之间复制数据的局限性，统一的分布式数据流框架得到了发展，以支持批处理和流处理。朝着这个方向，最新的机器学习和 AI，这些系统还公开了统一的编程接口，以无缝表达流和批处理应用程序

  * 下一步：As the next step in this unification, users have begun to combine latency-sensitive stream jobs with latency-tolerant batch jobs in a single application [19, 34]. We term these applications “stream/batch” where “stream” refers to the latency-sensitive jobs of the application and “batch” to the remaining ones. Examples of such applications can be found in the domains of online machine learning, real-time data transformation and serving, and low-latency event monitoring and reporting.

    一个具体的例子：Consider the stream/batch application in Figure 2, which imple- ments a real-time detection service for malicious behavior at an e-commerce company [51]. A batch job trains a machine learning model using historical data, and a stream job performs inference to detect malicious behavior. The stream/batch application therefore allows for the jobs to share application logic and state (e.g., using the same machine learning model between training and inference). This sharing facilitates application development and management. In addition, combined cluster resources can be used more efficiently by the different job types.

    

* Solution

  * The stream jobs of these applications must be executed with minimum delay to achieve low latency, which means that batch jobs may have to be preempted.（在统一流/批处理之前，必须将流和批处理作业分别部署在群集中，并通过通用资源管理器（如YARN [56]或Mesos）相互协作，这肯定会导致一些问题：资源利用率低下，由于流处理程序的优先级导致 batch 被干掉；或者超长的抢占时间）
  * Given that batch and stream jobs share the same runtime, our key idea is to **employ application-specific mechanisms and policies that dynamically prioritize stream jobs without <u>unduly penalizing</u>(过度的处罚) batch jobs**. 
  * Neptune, an execution framework for stream/batch applications that dynamically prioritizes tasks to achieve low la- tency for stream jobs. 
  * Neptune employs coroutines as a l**ightweight mechanism for suspending tasks without losing task progress**. It couples this fine-grained control over CPU resources with a **locality-and memory-aware** (LMA) scheduling policy to determine which tasks to suspend and when, thereby sharing executors among heterogeneous jobs.
  * **Lightweight suspendable tasks**. To prioritize latency-sensitive tasks, Neptune suspends tasks that belong to batch jobs on-demand.（最具启发意义的按需挂起） As a lightweight task preemption mechanism, **Neptune uses coroutines, which avoid the overhead of thread synchronization**. **Coroutines can suspend batch tasks within milliseconds, thus reducing head-of-line blocking for latency-sensitive tasks. When tasks are resumed, they preserve their execution progress**. To the best of our knowledge, Neptune is the **first distributed dataflow system** to use **coroutines** for task implementation and scheduling.
  * **Locality/memory-aware scheduling policy**. To satisfy the requirements of stream/batch applications when they **share the same executors**, Neptune uses a locality- and memory-aware (LMA) scheduling policy for task suspension. It load-balances by equalizing the number of tasks per node, thus reducing the number of preempted tasks; it is locality-aware by respecting a task’s prefer- ences; and minimizes memory pressure due to suspended tasks by considering real-time executor metrics (§5).
  * **Spark-based execution framework**. Neptune is designed to natively support stream/batch applications in Apache Spark [61], one of the most widely-adopted distributed dataflow frameworks. Users express stream/batch applications using the existing unified Spark API and add only one configuration line to indicate jobs as “stream” or “batch”. The implementation of Neptune is available as open-source1 (§6).

### Supporting Unified Applications

#### 2.1 Distributed dataflow platforms

Application programming interface.： a unification of APIs. This allows users to express both stream and batch computation through a single API, such as Spark’s Structured Streaming API [12], Flink’s Table API [3], or the external API layer of Apache Beam

执行模型：DAG，每个节点提供 executor，每个 executor 提供 slots，每个 slot 执行一个任务。An application scheduler assigns tasks to executors, respecting data dependencies, resource constraints and data locality

Evolution of execution model. ：原来那种一分为二的模型，必须部署多个框架，可能还有资源的多种切割

#### 2.2 Hybrid stream/batch applications

Users started combining latency-sensitive stream and latency-tolerant batch jobs as part of the same applica- tion. 

使用统一的设计意义是显而易见的，代码重用，避免内部系统间交互的复杂，有一个单一的应用监控和调参

To support stream/batch applications, an execution engine **must combine the different requirements of stream and batch jobs regarding latency2 and throughput, while achieving high resource utilization.** 

这个例子的特点

In Listing 1, the output of the latency-sensitive streaming job must be computed with low latency to ensure fast reaction times; the latency-tolerant batch job must run with high throughput to efficiently utilize the remaining resources and re- flect the most recent state in the model, as the training data is continuously updated.

阐述思考：When all resources are occupied, a streaming job may have to wait for other tasks to finish before starting processing, which leads to increased latency due to the queuing delay. Therefore, a platform for stream/batch applications must minimize queuing delays for latency-sensitive jobs, even under high resource utilization. At the same time, the throughput of batch jobs should not be compromised. Therefore, resources must be shared across jobs depending on their needs, while avoiding starvation and maintaining high utilization. **A unified execution engine must ensure high throughput and resource utilization despite the heterogeneity of job types.**

#### 2.3 Scheduling alternatives

图三这种情况下，理想是 stream 立马跑，剩下资源好好跑 batch，这就有各种各样的方法

基本方法：静态资源分配，每个 job 都使用专用的资源集。这对应于使用单独的引擎进行流和批处理，或使用单个引擎但为每个作业提交单独的应用程序的方法。 它是使用通用资源管理器并具有专门用于延迟关键任务的专用资源队列的现有生产环境中的常见解决方案。

FIFO：unified stream/batch engines such as Spark [13] and Flink [16] 。先来先得，batch 先来，吃掉所有资源，然后才是流处理，这导致流处理延迟很高

FAIR：大家都有权重，共享资源。总时间的确很低，但是不尊重低延迟应用需求

KILL：保证了延迟低，代价显而易见，要重启被干掉的 batch 作业。那资源利用率其实很差。即使 job 支持保留进度的抢占，也必须考虑检查点 task 状态的额外开销。

NEPTUNE：挂起策略，无任何检查点开销。

图4，耍流氓，因为抢占策略，不可能高的，你放个 kill 也是这样的

### DESIGN

To reduce the queuing of latency-sensitive tasks during execution, Neptune uses ***suspendable tasks***, which employ efficient **coroutines** to yield and resume execution. Neptune uses **a centralized scheduler** to assign application tasks to executors, which periodically send metrics back to the scheduler in the form of heartbeats. The **scheduler enacts policies that trigger task suspension and resumption**, which are ex- ecuted locally on each executor. The scheduling policies guarantee job requirements (see §2.2) and satisfy higher-level objectives such as balancing cluster load.

#### 3.1 Defining stream/batch applications

属性传递

#### 3.2 Scheduling stream/batch applications

* DAG 调度器计算每个 job 执行计划的 dag 图
* 构建执行顺序，相同任务分组并行计算，一个阶段全部完成以后下一阶段，stream 优先，剩余的 FIFO
* 任务到 task scheduler，然后具体执行，具有挂起策略，被挂起的不会被迁移，因为迁移会产生额外开销（**但是这一步似乎可以迁移**？解释不可以是因为流一般处理时间不长）

#### 3.3 Executing stream/batch tasks

*  Each executor has c cores and can run c concurrent tasks.
* 心跳，发送 executor 状态

### Suspendable Tasks

* 对用户透明，不需要用户去触发一个阶段性的 checkpoint
* 通常来说挂起这件事我们会想到线程，一些原语比如说 wait 和 notify，线程的同步需要系统调用，这件事情是很消耗资源的，很快就会成为瓶颈。
* 协程是用户空间线程，为了获得可扩展且高效的任务挂起机制，Neptune会使用堆栈式协程[47]。 协程是可恢复的函数，可以在计算内部的 yield point 处暂停执行。 常规函数可以被视为协程的特殊情况，协程不会中止执行，而是在完成后返回调用方。 当协程暂停执行时，它会将 status handle 返回给调用方，以后可将其用于恢复执行。
* Listing2示例代码，图6示意流程
* Although suspendable tasks are both efficient to sus- pend/resume and transparent to users, they keep state in memory, which may increase memory pressure on executors. however, that typical production clusters tend not to be constrained by memory.（这篇论文认为内存不是稀缺资源）

### LMA scheduling policy

Batch tasks run immediately as long as there are enough free resources, otherwise they wait for resources to become available. For stream tasks, Neptune’s scheduler uses the LMA (locality- and memory- aware) scheduling policy, outlined in Algorithm 1.

LMA 考虑任务的地域性偏好和执行器内存条件，避免 executors 内存压力（内存使用，磁盘溢出，垃圾回收活跃度）

为了防止低优先级的作业无限期地被暂停，Neptune还具有一个反饥饿机制（从伪代码中省略）。 对于每个任务，海王星都会跟踪其暂停的次数。 如果任务已暂停超过给定次数，则该任务将不间断运行一段时间，以确保每个阶段的进度。 请注意，可以使用相同的机制以及特定于应用程序的知识来限制重要批处理任务（例如更新共享状态的任务）引起的延迟。

### 实现

Spark includes a Driver that runs on a master node and an Executor on each worker node, all running as separate processes. The Executor uses a thread pool for running tasks. An application consists of stages of tasks, scheduled to run on executors. Every application maintains a unique SparkContext, the entrypoint for job execution.