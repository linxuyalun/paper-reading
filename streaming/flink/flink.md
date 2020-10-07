* [Apache Flink论文简读](https://blog.csdn.net/lpstudy/article/details/83661945)

### 2 System Architecture

1. 软件栈

   如上面图1所示，Flink Core底层是流处理引擎，然后在上层抽象出Batch processing和Stream
   Processing，再上层构建几种常用的应用：表格，图，机器学习，复杂事件处理等

2. 流处理模型

   客户端接收程序代码，转换成data-flow graph，然后提交给Job Manager。Job Manager将job分发给Task manager并跟踪状态和执行结果，发生失败的failover等等。

### 3 Streaming Dataflows

#### Dataflow Graphs

如图3所示的数据流图是有向无环图（DAG），它由以下各项组成：（i）stateful operators 和（ii）表示由 operator 生成的数据并且可供 operator 使用的数据流。 由于数据流图是以数据并行方式执行的，因此将 operator 并行化为一个或多个并行实例（称为子任务），并将流拆分为一个或多个流分区（每个产生子任务一个分区）。 在特殊情况下可能是无状态的stateful operator 实现所有处理逻辑（例如，过滤器，哈希联接和流窗口函数）。 这些 operator 中的许多是众所周知算法的教科书版本的实现。 在第4节中，我们提供有关窗口 operator 实现的详细信息。 流以各种模式（例如点对点，广播，重新分区，扇出和合并）在生产方和消费方运营商之间分配数据。

#### Data Exchange through Intermediate Data Streams

Flink的中间数据流是 operator 数据交换的核心抽象。中间数据流表示 operator 生成的数据的逻辑句柄，并且可以由一个或多个 operator 使用。中间流从逻辑上讲是合理的，因为它们指向的数据可能会或可能不会在落盘。数据流的特定行为由Flink中的高层参数化（例如，DataSet API使用的程序优化器）。

Pipelined and Blocking Data Exchange。流水线中间流在同时运行的生产者和使用者之间交换数据，从而导致流水线执行。**结果，流水流将反压从消费者传播到生产者，并通过中间缓冲池对某些弹性进行模运算，以补偿短期的吞吐量波动**。（见[反压的含义及其解决策略](https://www.jianshu.com/p/bae101ec7bcf)） Flink将流水线流用于连续流程序以及批处理数据流的许多部分，以尽可能避免实现。另一方面，阻塞流适用于有界数据流。阻塞流在使生产 operator 的数据可供使用之前会缓冲所有生产 operator 的数据，从而将生产 operator 和消费 operator 分成不同的执行阶段。阻塞流自然需要更多的内存，经常溢出到辅助存储，并且不会传播背压。它们用于将连续的 operator 彼此隔离（在需要时），以及在使用管道中断 operator 的计划（例如，合并合并联接）可能导致分布式死锁的情况下。

平衡延迟和吞吐量。 Flink的数据交换机制是围绕缓冲区交换实现的。在生产者端准备好数据记录后，它会被序列化并拆分为一个或多个缓冲区（一个缓冲区也可以容纳多个记录），这些缓冲区可以转发给使用者。 i）缓冲区已满或ii）达到超时条件时，会将缓冲区发送给使用者。通过将缓冲区的大小设置为较高的值（例如几千字节），Flink可以实现高吞吐量，而通过将缓冲区的超时时间设置为较低的值（例如几毫秒）可以实现低延迟。图4显示了缓冲区超时对在30台计算机（120个内核）上的简单流grep作业中传递记录的吞吐量和延迟的影响。 Flink可以实现20ms的99％的可观察到的延迟。相应的吞吐量是每秒150万个事件。随着我们增加缓冲区超时，我们看到延迟随着吞吐量的增加而增加，直到达到完整的吞吐量（即，缓冲区的填充速度快于超时期限）。在50毫秒的缓冲区超时下，群集达到每秒超过8000万个事件的吞吐量，而50毫秒的延迟为99％。

控制事件。除了交换数据外，Flink中的流还传递不同类型的控制事件。这些是特殊事件，由 operator 注入到数据流中，并与流分区中的所有其他数据记录和事件一起按顺序传递。接收 operator 在到达时通过执行某些操作来对这些事件做出反应。 Flink使用许多特殊类型的控制事件，包括：

* *checkpoint barriers* that coordinate checkpoints by dividing the stream into pre-checkpoint and post- checkpoint (discussed in Section 3.3),
* *watermarks* signaling the progress of event-time within a stream partition (discussed in Section 4.1),
* *iteration barriers* signaling that a stream partition has reached the end of a superstep, in Bulk/Stale-Synchronous-Parallel iterative algorithms on top of cyclic dataflows (discussed in Section 5.3).

如上所述，控制事件假定流分区保留记录的顺序。为此，Flink中消耗单个流分区的一元 operator 可保证记录的FIFO顺序。但是，接收多个流分区的 operator 会按到达顺序合并流，以跟上流率并避免背压。结果，**在任何形式的重新分区或广播之后，Flink中的流数据流都不提供顺序保证，并且处理无序记录的责任留给了 operator 实施**。我们发现，这种安排提供了最有效的设计，因为大多数 operator 不需要确定性的顺序（例如，哈希联接，映射），并且需要补偿无序到达的 operator （例如事件时间窗口）可以作为 operator 逻辑的一部分，可以更有效地执行此操作。

#### 3.3 容错

见另一篇主论文

Apache Flink的检查点机制建立在分布式一致快照的概念上，以实现精确的一次处理保证。数据流的可能无限制的性质使恢复时的重新计算变得不切实际，因为长时间运行的工作可能需要重演数月的计算。为了限制恢复时间，Flink会对 operator 的状态进行快照，包括按固定间隔输入流的当前位置。

核心挑战在于，在不停止执行拓扑的情况下，对所有并行 operator 进行一致的快照。本质上，所有 operator 的快照在计算中应引用相同的逻辑时间。 Flink中使用的机制称为异步 barrier 快照（ABS [7]）。 barrier 是注入到输入流中的控制记录，这些控制记录对应于逻辑时间，并在逻辑上将流分离为效果将包含在当前快照中的部分和稍后将被快照的部分。

 operator 从上游接收 barrier ，然后首先执行对齐阶段，确保已收到所有输入的 barrier 。然后， operator 将其状态（例如，滑动窗口的内容或自定义数据结构）写入持久性存储（例如，存储后端可以是外部系统，例如HDFS）。备份状态后， operator 会将 barrier 向下游转发。最终，所有 operator 都将注册其状态的快照，并且将完成全局快照。例如，在图5中，我们显示快照t2包含所有 operator 状态，这些状态是消耗t2 barrier 之前的所有记录的结果。 ABS与用于异步分布式快照的Chandy-Lamport算法相似[11]。但是，**由于Flink程序的DAG结构，ABS无需检查飞行中的记录，而仅依靠对齐阶段将其所有效果应用于 operator 状态**（这里没看懂，等后续另一篇文章）。这保证了将需要写入可靠存储的数据保持在理论上的最小值（即仅 operator 的当前状态）。

从故障中恢复后，所有 operator 状态都将恢复为上次成功拍摄快照所采取的相应状态，并从有快照的最新 barrier 开始重新启动输入流。恢复时需要的最大重新计算量被限制为两个连续障碍之间的输入记录量。此外，通过额外重播在直接上游子任务处缓存的未处理记录，可以部分恢复失败的子任务[7]。

ABS具有几个好处：i）它可以保证一次状态更新，而不会暂停计算； ii）它与其他形式的控制消息完全分离（例如，通过触发窗口计算并因此不限制窗口的事件） iii）完全与用于可靠存储的机制分离开来，从而可以将状态备份到文件系统，数据库等，具体取决于使用Flink的较大环境。

>  4 5 6 7 不再看了，主要是关于 stremaing 抽象的一些想法和批处理的兼容。