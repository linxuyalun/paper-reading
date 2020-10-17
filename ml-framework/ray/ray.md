前置知识：

* [监督学习 – Supervised learning](https://easyai.tech/ai-definition/supervised-learning/)
* [深度强化学习总结](https://www.jianshu.com/p/d5aac10f4517)
* [Ray: A Distributed Framework for Emerging AI Applications 译文](https://blog.csdn.net/u011254180/article/details/79327639)

下一代AI应用程序将不断与环境交互，并从这些交互中学习。这些应用程序在性能和灵活性方面提出了新的和苛刻的系统要求。我们常讨论的模型都是监督型学习，而实际上，越来越多的应用程序不是做出和服务于单一预测，而是必须越来越多地在动态环境中运行，对环境变化作出反应，并采取一系列行动以实现一个目标。这些更广泛的要求自然地在强化学习（RL）的范式内进行，该学习在不确定的环境中学习持续运行。比如 Alpha Go，自动驾驶，无人机。

一个简单的例子，强化学习中，agent 通过不断的试错来探索最佳的行动策略。所谓的“强化”（reinforcement）是指通过奖励（或惩罚） 让 agent  在给定状态下更加倾向于（或避免）做某一动作，这是一种反馈强化的过程。Agent 根据当前的环境 state 给出一个动作 action，模拟器会根据对应的动作返回相应的 reward，可能是奖励可能是惩罚，agent 再根据以往的 state 和 reward 以及当前的 state 和 reward 改进模型，生成心的策略，并做出下一步动作。

RL 有两个特点，一个是 **reward delay**，动作与收益之前的对应关系可能比较复杂，或者有明显延迟。例如在非常经典的"太空入侵者"游戏中，只有“开火”这个动作会消灭敌人，带来收益，但是“向左移动”或“向右移动”这些动作也是很关键的，尽管不会带来直接收益。在有些时候需要放弃短期收益，来获取更大的长期累计收益；一个是 **balance between exploration and exploitation**：是在现有的比较好的 policy 上继续深挖，还是跳转到全新的 policy 上去碰碰运气。现实生活中也存在类似的权衡：研究人员是进入一个全新的、更有前景的研究领域，还是坚守已有丰硕成果的熟悉的领域。

有以下几个特点将 RL 应用与传统的监督学习应用区分开来：

* 首先，他们经常严重依赖模拟来探索状态和发现行动的后果。模拟器可以编码计算机游戏的规则，诸如机器人的物理系统的牛顿动态，或者虚拟环境的混合动力学；

* 这同样意味着大量计算，一个现实的应用程序可能会执行数以百万计的模拟。可以想象，在机器人与物理环境进行交互的情况下，我们需要推断环境的状态并在几毫秒内计算新的行为。同样，模拟也可能需要几毫秒的量级。因此，我们需要能够在不到一毫秒的时间内安排任务。否则，调度开销可能会很大；

* 而且不同的模型它的持续时间差很多，计算轨迹花费时间差异很大。比如一个游戏，可能几步就输掉了，几百步才能获胜；

* RL 应用程序的图计算是异构的并且是动态演化的。什么意思呢，对于现在流行的计算框架而言，实现的都是批量同步并行（BSP）模型，使用 BSP，同一阶段内的所有任务通常执行相同的计算（尽管在不同的数据分区上）并且花费大致相同的时间量。而对于动态图而言，只要部分 rollouts 完成，就会启动新的 rolleouts 以维护执行 rolleouts。这使得执行图形具有动态性，因为我们无法预测 rollouts 的完成顺序或哪些 rollouts 将用于特定策略更新；

* 最后当然还要易于开发。

所以，我们对新型的 ML 框架系统的要求是：

* Need to handle dynamic task graphs, where tasks have

* - Heterogeneous durations
  - Heterogeneous computations

* Schedule millions of tasks/sec

* Make it easy to parallelize ML algorithms written in Python






