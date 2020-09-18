* Issue

  The rise in popularity of **machine learning, streaming, and latency- sensitive online applications** in **shared production clusters** has raised new challenges for cluster schedulers. To optimize their performance and resilience, **these applications require precise control of their placements, by means of complex constraints**, e.g., to collocate or separate their long-running containers <u>across groups of nodes</u>. 

  Since cluster managers are application-agnostic, they have enabled cluster operators to consolidate diverse workloads onto shared clusters.

  In the presence of these applications, the cluster scheduler must attain global optimization objectives, **such as maximizing the number of deployed applications or minimizing the violated constraints and the resource fragmentation, but without affecting the scheduling latency of short-running containers**.

  At the same time, **placing LRAs, along with batch jobs, in shared clusters is appealing to reduce cluster operational costs, avoid unnecessary data movement, and enable pipelines involving both classes of applications**.

  Despite these observations, support for LRAs in existing sched- ulers is rudimentary [2, 31, 33, 53]. In particular, the bespoke scheduling requirements of LRAs (§2) remain largely unexplored (§8): (i) Precise control of container placement is key for optimizing the performance and resilience of LRAs. Simple affinity and anti-affinity constraints separate them to minimize resource interference or chance of fail- ure), which are already partially supported by a few schedulers, are necessary but not sufficient. Our experiments with various LRAs (e.g., HBase, TensorFlow, and Storm; see §2) reveal that **powerful constraints are required to capture interactions between containers and unlock the full potential of LRAs**.

  (ii) When placing LRA containers, the cluster scheduler must achieve global optimization objectives, such as minimizing the violation of placement constraints, the resource fragmentation, any load imbalance, or the number of machines used. Due to their long lifetimes, LRAs can tolerate longer scheduling latencies than tra- ditional batch jobs. The second requirement for LRA placement is therefore to allow cluster operators to **optimize for global cluster objectives, but without impacting the scheduling latency of short-lived containers.**

* Solution

  * Our experiments with various LRAs (LONG RUNING APPLICATION e.g., HBase, TensorFlow, and Storm; see §2) reveal that powerful constraints are required to capture interactions between containers and unlock the full potential of LRAs.
  * Due to their long lifetimes, LRAs can tolerate longer scheduling latencies than tra- ditional batch jobs. The second requirement for LRA placement is therefore to allow cluster operators to optimize for global clus- ter objectives, but without impacting the scheduling latency of short-lived containers.
  * A new cluster scheduler designed for the **placement of long- and short-running containers**. 
  * Medea introduces **powerful placement constraints** with formal semantics to capture interactions among containers within and across applications.
  * It follows a novel two-scheduler design: (i) for long-running containers, it applies an **optimization-based approach that accounts for constraints and global objectives**; (ii) for short-running containers, it uses **a traditional task-based scheduler for low placement latency**. 
  * **Expressive, high-level constraints**. Medea enables application owners and cluster operators to specify powerful placement constraints across LRA containers with formal semantics. **Relying on the notions of container tags and node groups**, Medea supports both intra- and inter-application constraints, without requiring knowledge of the cluster’s configuration or of already- deployed LRAs
  * Evaluated on a 400-node cluster, our implementation of Medea on Apache Hadoop YARN achieves placement of long-running applications with significant performance and resilience benefits compared to state-of-the-art schedulers.
  * shared production cluster ？？

### Medea

**A new cluster scheduler that enables the placement of both long- and short-running containers.** 

* **Two-scheduler design**. 
  * A dedicated scheduler for the placement of LRAs, task-based applications are scheduled directly by a traditional scheduler. This design ensures that the scheduling latency for task-based applications is not impacted, while enabling high-quality placements for LRAs.
  * It also makes Medea compatible with existing task-based schedulers, reusing existing scheduler implementations and facilitating adoption in production settings (§3).
* **Expressive, high-level constraints.** 
  * Medea enables application owners and cluster operators to specify powerful placement constraints across LRA containers with formal semantics. 
  * Relying on the notions of **container tags and node groups**, Medea supports both intra- and inter-application constraints, without requiring knowledge of the cluster’s configuration or of already- deployed LRAs (§4).
* **LRA scheduling algorithm**. 
  * A placement of LRAs with constraints as an integer linear program (ILP), and solve it as an online optimization problem.
  *  Unlike existing approaches, our algorithm considers multiple LRA container requests at once to achieve higher-quality placements and global objectives (§5). 
  * We investigate heuristics that trade placement quality for lower scheduling latency.
* 评估：Medea reduces median runtime of HBase and TensorFlow work- loads by up to 32% compared to our implementation of Kubernetes’ scheduling algorithm (J-Kube) and by 2.1× compared to YARN, while significantly reducing runtime variability too. Moreover, it improves application unavailability by up to 24% compared to J-Kube. Medea leads to constraint violations of less than 10% even for complex inter-application constraints involving 10 LRAs, and to reasonable scheduling latencies. Finally, it does not affect neither the schedul- ing latency nor the performance of task-based jobs.

### LONG-RUNNING APPLICATIONS IN CLUSTERS

* 2.1 Use Cases

* 2.2 Application Performance

  * Affinity：Both intra- and inter-application affinity constraints are crucial to unlock full application performance.

  * Anti-affinity. To minimize resource interference between LRAs, it may be desirable to place containers on different machines through intra- and inter-application anti-affinity.

  * Cardinality. The affinity and anti-affinity constraints represent the two extremes of the collocation spectrum. To strike a balance between the two, we experiment with more flexible cardinality con- straints, which set a limit on the number of collocated containers.

    Based on these results, we make the crucial observation that **affinity and anti-affinity constraints, albeit beneficial, are not sufficient**, and tighter placement control using cardinality constraints is required. In our experiments in the highly utilized cluster, we observe that collocating up to 16 TensorFlow workers on a node reduces runtime by 42% compared to the affinity placement (maxi- mum cardinality of 32), and by 34% compared to the anti-affinity placement (maximum cardinality of 1). A second observation is that the cardinality that leads to optimal runtimes **can vary based on the specific application and the current cluster load**. Indeed, in the experiment of Figure 2d, the optimal cardinality value is 16 for the highly utilized cluster and 4 for the the less utilized one.

* 2.3 应用恢复

  * With a random placement, an application may lose multiple containers at once, which can in turn impact its recovery time or performance. Such a placement scheme hurts LRAs in particular because their containers are by definition long-lived and thus the failure probability increases over time. Hence, application owners must be able to spread containers across fault and upgrade domains (anti-affinity constraints).
  * However, it should not be required to explicitly refer to spe- cific node domains when requesting containers: (i) these domains change over time, e.g., with node addition/removal; and (ii) it is cumbersome to enumerate all domains of a cluster with thousands of machines. In cloud environments, it may even not be feasible— the operator may not reveal the cluster configuration for security and business reasons.

* 2.4  Global cluster objectives：Cluster operators must also specify constraints that guarantee the smooth operation of the cluster. Besides local constraints, such as restricting the number of network-intensive containers per node, the scheduling of LRAs must meet *global objectives*:

  * Minimize constraint violations.
  * Minimize resource fragmentation. 
  * Balance node load. 
  * Minimize number of machines used. 

  some objectives may be conflicting, e.g., minimizing con- straint violations and load imbalance, while others may be irrele- vant in a specific scenario, e.g., minimizing the number of machines for an on-premises cluster. The cluster operator should be able to determine the objectives to be used and their relative importance. Supporting such global objectives should not affect the scheduling of traditional batch jobs, which are more sensitive to container allocation latencies due to their shorter container runtimes.

* 2.5 Scheduling Requirement （effective scheduling of LRAs）

  * [R1] **Expressive placement constraints**: We must support intra- and inter-application (anti-)affinity and cardinality to express de- pendencies between containers during placement.
  * [R2] **High-level constraints**: The constraints must be high-level, i.e., agnostic of the cluster organization, and capable of referring to both current and future LRA containers.
  * [R3] **Global objectives**: We must meet global optimization objec- tives imposed by the cluster operator.
  * [R4] **Scheduling latency**: Supporting LRAs, which can tolerate higher scheduling latencies, must not affect the scheduling latency for containers of task-based applications.
  * Most schedulers support (only intra-application and non-high-level) constraints implicitly through machine attributes (e.g., place two containers on a node with a given hostname or on machines with GPUs). **Only Kubernetes supports explicit intra- and inter-application high-level constraints between containers, but not cardinality, and lacks full support for global objectives by considering only one container request at a time.** More details are provided in §8.

### Medea Design

设计图 Figure4

Medea supports **the scheduling of both LRAs and “traditional” applications with short-running containers (referred to as task- based jobs)**. As shown in Figure 4, Medea uses a two-scheduler design for placing containers: (i) a dedicated LRA scheduler places long-running containers, accounting for constraints stored in the constraint manager component; and (ii) a task-based scheduler places task-based jobs.

**LRA INTERFACE**

**The LRA scheduler uses an online optimization- based algorithm that, given the current cluster condition, including already running LRAs and task-based jobs, determines the efficient placement of newly submitted LRAs**. The scheduler is invoked at regular configurable intervals to place all LRAs submitted during the latest interval. Our scheduling algorithm, described in §5, takes into account multiple LRA container requests at once to satisfy their placement constraints and attain global optimization objectives (requirement R3).

**Task-based scheduler.** 

Medea’s two-scheduler design **removes the burden of handling complex placement decisions from the task- based scheduler**, allowing the scheduling latency for task-based jobs to remain low (**requirement R4**). Moreover, this design allows the reuse of existing production-hardened task-based schedulers. This minimizes the changes required in the existing scheduling infrastructure, **and is crucial for adopting Medea in production**.

**Constraint manager.**

We introduce a new central component in which all constraints—both from application owners and cluster operators—are stored. **This allows Medea to have a global view of all active constraints and to easily add or remove constraints.**

Medea operates as follows: it passes placement decisions made by the LRA scheduler (step 1 in Figure 4) to the task-based scheduler (step 2), which then performs the actual resource allocation (step 3). This approach **avoids the challenge of conflicting placements**, faced MEDEA Scheduler by existing multi-level [27, 45] and distributed [14, 30, 42, 44] sched- ulers: in these designs, different schedulers, operating on the same cluster state, may arrive at conflicting decisions, whereas in Medea the actual allocations are performed by a single scheduler. We fur- ther discuss placement conflicts in §5.4.（其他的实现可能会遇到各种冲突）

### DEFINING PLACEMENT CONSTRAINTS

#### Tag model and node groups

Container tags. 不止对机器打标签，对容器打标签

Tag sets. 一个 tag 集合，比如 Tn 表示节点 n 上的所有 tag 集合，因此是动态的，同理，这个 n 也可以表示几个节点的集合

Node Groups：逻辑层面的节点分类 node，在同一个物理机架上的 rack，fault，upgrade。这种 node 的标签可以让打标签的时候不用考虑去探测集群的状态

#### Placement constraints

pecify placement constraints using tags to refer to containers in the same or different LRAs, and node groups to target specific node sets. In particular, it supports constraints of the following form:

```
C = {subject_tag, tag_constraint, node_group}
```

Caf = {storm, {hb ∧ mem, 1, ∞}, node} 

表示所有 tag 为 storm 的容器与至少一个标签为 hb 和 mem 的容器放置在一个 node 域中

Ccg = {spark, {spark, 3, 10}, rack}

表示 tag 为 spark 的容器至少三个最多10个在同一个 rack 中

**约束依赖**：如果约束必须以另一个已部署应用程序的特定容器为目标，则需要其应用程序和容器ID。 为了解决此问题，应用程序所有者可以将两个应用程序一起提交。 否则，可以使用公开已部署的应用程序及其容器的服务。

**复合依赖：**此类约束以析取范式（DNF）的形式指定，允许约束的任意组合。

**软依赖和权重**：尽力满足，不强制。比如要求具有 node 和 rack 亲和性，前者权重更高。可以使用权重比来模拟硬依赖

### Scheduling LRAS

ILP-based (§5.2) and heuristic-based (§5.3) LRA scheduling algorithms

LRA 调度器参考三个内容：

* 容器需求以及放置约束策略；
* 满足最新提交的 LRAs 的放置策略，满足之前的放置策略，满足 cluster operator 的要求；
* 每个节点的可用资源。

#### ILP-based scheduling algorithm

整数线性规划问题。

#### 启发式算法

### 实现

见论文图

### 评估

TODO