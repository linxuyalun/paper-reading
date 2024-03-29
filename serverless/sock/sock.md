* [Docker技术原理之Linux UnionFS（容器镜像）](https://www.jianshu.com/p/3ba255463047)
* [docker的aufs笔记](https://blog.csdn.net/weixin_28738845/article/details/81810145)
* [DOCKER基础技术：AUFS](https://coolshell.cn/articles/17061.html)
* [runC容器和安全沙箱（runV）容器的区别](https://help.aliyun.com/document_detail/160309.html)
* [mount --bind和硬连接的区别](https://blog.csdn.net/shengxia1999/article/details/52060354)
* [SOCK：Serverless场景的runc容器启动优化方案](https://kernel.taobao.org/2019/06/SOCK-Rapid-Task-Provisioning-with-Serverless-Optimized-Containers/)

### 容器性能剖析

物理机如何提升容器文件系统（即挂载docker image）的密度？使用namespace进行资源隔离的开销有多大？哪些资源是必须隔离的？如何避免cgroup的创建开销？作者的实验环境：8-core m510机型，4.13.0-37内核。

#### 容器存储

Container 存储提供一个独立的根文件系统，隔离I/O资源。功能需求简单明了，而实现上则是百花齐放。已有的容器存储解决方案丰富(如overlay，AUFS，devicemapper，btrfs等)。本文作者以AUFS为例子。

像 AUFS 这种我们可以称为联合挂载，将不同目录挂载到同一个目录下面，只有最上层是可读可写，下层都是只读层。

当容器需要修改一个文件，而该文件位于低层branch时，顶层branch会直接复制低层branch的文件至顶层再进行修改，而低层的文件不变，这种方式即是CoW技术（写复制），AUFS默认支持Cow技术。

当容器删除一个低层branch文件时，只是在顶层branch对该文件进行重命名并隐藏，实际并未删除文件，只是不可见，这种方式即AUFS的whiteout（写隐藏）。

本文作者以AUFS方案作为基线，与bind mount进行对比。

Bind挂载允许当前文件系统的一个文件或者目录挂载在另一处目录下。

当mount —bind命令执行后，Linux将会把被挂载目录的目录项屏蔽，注意，只是隐藏不是删除，数据都没有改变，只是访问不到了。同时，内核将挂载目录的目录项记录在内存里的一个s_root对象里。

命令执行完后，当访问被挂载目录下的文件时，系统会告知目录项被屏蔽掉了，自动转到内存里找VFS，通过vfsmount了解到两个目录的对应关系，从而读取到挂在目录的inode，这样在被挂载目录下读到的全是挂载目录下的文件。

由上述过程可知，mount --bind 和硬连接的重要区别有：

1.mount --bind连接的两个目录的inode号码并不一样，只是被挂载目录的block被屏蔽掉，inode被重定向到挂载目录的inode（被挂载目录的inode和block依然没变）

2.两个目录的对应关系存在于***\*内存\****里，一旦重启挂载关系就不存在了

bind挂载是不支持COW语义的，一般而言可以用来实现不同容器间data volume共享。

> [linux 文件系统之 mount 流程分析](https://blog.csdn.net/wuruixn/article/details/9619127)

一旦填充了主机文件系统上的子目录，希望用作容器根目录，加下来就必须切换根目录并不访问其他主机文件数据。Linux 提供了两种用于修改容器可见文件系统的原语。

一种叫 chroot，针对某个进程，使用指定目录作为新的根目录，在新根下将访问不到旧系统的根目录结构和文件。

另一种是现在用的方法，unshare + 特定 mount 相关的参数。

当一个进程fork之后，整个进程表项被复制，包括所有的文件描述符。但是文件表项并不会被复制。父进程和子进程共享相同的文件表项。

unshare则是创建了新的mount namepsace，使用形态与 COW 类似。用户可以在新namespace里进行mount/unmount操作，修改只对当前namespace可见。灵活度很高，但同时也带来了性能问题。

图2展示了已有mount namespace的数量对新创建以及删除namesapce的影响。

随系统已有namespace数量增加，ops趋近于0，可以看到mount namespace的扩展性差。

而与之相比我们之前提到的 chroot 带来的开销几乎可以忽略不计，整个过程延迟都小于1微秒。

#### 逻辑隔离：namespace

上一小节已提到了mount namespace，那么这一小节则会涉及NET,UTS,IPC,PID namesapce。

Mount: 隔离文件系统挂载点

UTS: 隔离主机名和域名信息

IPC: 隔离进程间通信

PID: 隔离进程的ID

Network: 隔离网络资源

User: 隔离用户和用户组的ID

Cgroup: 资源间隔离

unshare允许用户创建以及切换新的namesapce，namespace类型由用户通过参数控制。当使用namespace的最后一个进程退出时，则销毁该namespace。不同数量的进程执行unshare以及退出，用ftrace来测量不同namespace类型的创建销毁开销。Fig3. 选取了开销top4的的namespace操作。 可以看到，mount 以及 IPC namespace的延迟大约在几十ms量级，通过调查发现，延迟主要是因为等待一个RCU grace period完成。

所谓RCU，Read-Copy Update，一种同步机制。

由于等待上下文并没有持锁，所以并不会对吞吐有影响。事实上，由前一小节也可得知，mount namespace的最高创建ops为～1500。

既然 net ns 影响这么大，继续测量了net ns对容器创建销毁的影响。如图所示，无任何优化时的吞吐是200c/s (containers/second)。通过disable IPv6以及移除比较耗时的广播逻辑，吞吐可达400c/s；如果完全不使用net ns，吞吐可达900c/s。

#### 性能隔离：cgroup

Linux的cgroup可以用来实现不同资源（如CPU，memory，blk I/O，net等）的隔离。主要的使用模式有两类，第一类分为四个步骤：1. 创建cgroup；2. 创建进程并attach到cgroup里；3. 进程退出；4. 删除cgroup。第二类则是通过cgroup复用，大部分场景只有步骤 2 && 3。

Fig.5对比了这两类使用模式的区别。可以看到cgroup复用相比于cgroup每次重新创建的方式，效率至少提升一倍。线程数=16时，吞吐达到峰值，这是因为系统的HT=16。

根据上述的几组观察以及实验，思考serverless的实现方案。在sererless场景，handlers可能仅依赖于一个或少量的基础镜像，所以union文件系统的弹性叠加特性并不是必须的，倾向于使用开销更小的bind挂载方式。同样的，可以使用开销小的chroot来替代mnt ns。serverless平台跑的并非service后台服务，端口静态绑定并非是必选项，所以net ns也可不使用。最后，cgroup复用对降低延迟或者提升吞吐都很有意义。

### Lean Container

有了这些前置知识，这部分内容就很好理解了。

SOCK使用bind mount 将 host的四个目录合并成为容器的根目录，带’F’标志的目录。

所有容器的 base 一样，都是ubuntu系统，将它放置在内存中。

Packages目录用于缓存包，所有容器共享，后面会讲到。

lambda code目录(只读)与 scratch 目录(可写)是私有权限。

接下来就是切换根目录，目录合成后，使用chroot进行根目录切换，并创建 init 与 helper 两个进程。

后续所有的 children 都是继承了这个 root

init 进程是 sock container 要跑的第一个进程，它就调用了 unshare，但是不包括 net 和 mnt 这两个namespace 的创建

既然不包括网络 namespace，那么它是怎么通信的呢，如图，scratch目录包含了一个Unix domain socket（它可以用于进程间通信），这里用于OpenLambada manager 与 容器内的进程通信。该通道还被用于控制平面，如一些特权控制。

Linux 进程使用cgroup与namespace隔离。由于cgroup创建的开销较大，SOCK使用了cgroup pool 进行优化，容器创建时直接从pool中申请，容器销毁后则将cgroup释放给pool。

### Python初始化开销分析

Python里主流的package是哪些？这些package的初始化开销有多大？在本地的lambda worker中，缓存主流的packages是否具有可行性？

作者对github上876K个python项目进行包依赖分析，选取了最主流的top20 packages。其中36%的import集中在这20个包(对应PyPi源所有packages的0.02%)。这20个包可分为五类：网络框架，数据分析，通信，存储，开发。其中，网络框架类可能会被基于serverless的网络框架替换，开发类则还没有应用场景。

第一次使用某个package，需要三个步骤：download，install，import。继续使用该package，则可以跳过其中某些步骤。top20 包的初始化开销如Fig.7所示，平均初始化时间为1～13s，细分成三步：download开销1.6s，install开销2.3s，import开销107ms。

接下来分析将PyPi仓库存储在本地的可行性。PyPi仓库包含101K个packges，Fig 8. 显示了整个仓库的packages分布，不包含索引文件，总大小为～1.5TB，压缩后～0.5TB。

接下来需要回答，有多少PyPI packages可以共存安装在本地。这里不描述细节，直接给出结论，约97%的包是可以共存的。

### Generalized Zygotes

这里的核心问题是，尽管我们已经清楚解决方案了，可是我到底该怎么 pre-import。

或者说，我怎么让那些已经“热”起来的包在你的容器里执行。

来看一下这里的解决方法。

首先，每个进程的 ipc 是设置好的，所以可以进程间通信。

我想要知道你这个容器的 namespace，知道你这个容器的 root file system，我核心需要知道什么。

文件描述符，图里的 fd（file descriptors），每一个进程所维护的该进程打开文件的记录表。

manager 会把这个 fd 传给 Zygotes 的 helper 函数，Zygotes 也是在容器里的，然后 Zygotes 会 fork 一个子进程，叫做 tmp。

这个进程随后会执行一个叫做 fchdir 的命令，它接受一个参数，就是我们传进去的 fd，它就把当前的工作目录变成 fd 所有的工作目录。然后执行 chroot，修改根目录。

这个进程还会执行 setns 的命令，那就是之前干的事情，改各个 namespace，比如 ipc，pid，user，uts 这些

Setns 的一个特殊之处是，在调用之后，仅部分应用于调用者的所有命名空间。因此，需要再次 fork，创建了 grandchild helper，这个 helper 就是在容器 H 中完全运行的了。

这里还要注意的一个细节是，Zygote 本身会创建多个 Zygote，初始的 Zygote 什么预加载包都没有，新生成的 Zygote 都是从已有的 Zygote 创建出来的，不同的 Zygote 包含的预加载的包不同。从而这整个形成了一个树，因为都是 fork 出来的，所有 zygotes 之间的页的共享可以降低内存消耗。

这样子的 lambda 加载对应的 pre-import 的 zygote。

那为什么不直接所有主流包直接放在一起呢？论文解释说主要为了避免恶意包的存在，不同的 lambda 按需获取。

### Serverless Cache

SOCK使用了三层缓存：handler & install & import缓存，如Fig.12 所示。handler 缓存包含了未使用的handler容器，这些容器均处于暂停状态。它们不消耗CPU，但消耗内存，使用LRU淘汰算法来限制内存开销。install 缓存指预先在磁盘上安装的packages(占全量的97%)，已安装的包通过只读方式挂载至每一个容器。import缓存主要是用来管理Zygotes。Zygotes会消耗内存，且package的使用频率随时间而变化，所以Zygotes的管理相对复杂一些。特别是，当内存不够时，它的逐出策略会比较复杂。特别是它们之间的共享内存使得很难解释特定 Zygote 占用比。论文说它用了启发式的算法来估算内存使用和驱逐收益，但是很遗憾的是论文并没有在此问题上深入的写，也没有相关公式，所以就没法详细解释了。

### 总结

SOCK是由若干优化手段叠加而成的一套容器优化方案，其中缓存加速本质是通过空间(本地磁盘或者内存)换时间(低延迟高吞吐)，优化理念也并非完全新鲜(安卓的Zygote，AWS的container reuse，kata的vmtemplate等)。SOCK的价值在于对这些优化理念进行了加工，整合形成一套适用于runc平台下，为Serverless服务的容器解决方案。