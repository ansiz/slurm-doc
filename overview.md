# 概览 
Slurm是一个可工作于各种不同规模的Linux集群之上的开源、具备容错性和高度可扩展性的集群管理和作业调度系统。Slurm的操作不需要内核上的修改且相对独立。作为一款集群作业管理系统，Slurm包含三大主要功能：

1. 在特定时间段内为用户分配对资源（计算节点）的独占和/或非独占访问权限，以便他们可以执行作业。
2. 它提供了在分配的节点上启动、执行和监控作业（通常是并行作业）的框架。
3. 通过管理待处理作业的队列来仲裁资源的争用。  

除此之外还提供了可选的插件用于计费、高级预约、组调度、回填调度、拓扑优化的资源选择、基于用户或账户的资源限制以及复杂的多因素作业优先级算法等功能。

## 架构
Slurm有至少一个集中管理器`slurmctld`用于监控资源和作业，也可能存在备用的`slurmctld`用于系统故障后的责任接管。每个计算节点上都有一个名为`slurmd`的守护进程，类似于一个远程shell，它不断地等待作业分配、执行作业、返回作业状态，`slurmd`提供了容错的分层通信。可选的`slurmdbd`(Slurm DataBase Daemon) 用于在单个数据库中记录多个Slurm管理的账务信息。  

还包含一些工具命令，其中用户工具包括：  
* srun：用于创建作业
* scancel：用于终止排队等候或正在运行的作业
* sinfo： 用于报告系统状态
* squeue： 用于报告作业状态
* sacct：用于获取正在运行或已经完成的作业或作业步信息
* smap和sview： 以图形方式报告系统和作业状态， 包括网络拓扑  

管理员工具包括：
* scontrol： 用于监视和/或修改集群上的配置和状态信息。
* sacctmgr： 数据库管理工具用于识别群集，有效用户，有效的帐户等信息。

![Slurm components](http://o6sdpimwf.bkt.clouddn.com/uploads/2017/09/arch.gif)  

Slurm包含了调度所需的各种功能，除基本功能之外Slurm还提供了可用于轻松支持各种基础设施的通用插件机制，这允许模块化的方式配置Slurm，下面是目前支持的插件以及它们的功能：  
* Accounting Storage： 主要用于存储有关作业的历史数据。当与`slurmdbd`一起使用时，它还可以提供基于限制的系统以及历史系统状态。
* Account Gather Energy： 收集系统中每个作业或节点的能耗数据，此插件集成于`Accounting Storage`和`Job Account Gather`插件中。
* Authentication of communications: 提供Slurm各种组件之间的认证机制。
* Checkpoint: 提供检查机制接口
* Cryptography (Digital Signature Generation)： 用于生成数字签名的机制，验证作业步被授权在特定节点上执行。这与用于验证的插件不同，因为作业步请求是从用户的srun命令而不是直接从slurmctld守护进程发出的，该插件用于生成作业步凭证及其数字签名。
* Generic Resources：提供用于控制如GPU、CPU等通用资源的接口。
* Job Submit： 自定义作业提交插件。
* Job Accounting Gather： 收集作业步资源使用情况。
* Job Completion Logging： 负责作业终止记录，所收集的数据通常是` Accounting Storage Plugin`的子集。
* Launchers： 控制`srun`命令用于启动任务的机制。
* MPI： 为各种MPI实现提供不同的钩子。例如用于设置MPI特定的环境变量。
* Preempt： 设定作业抢占机制。
* Priority:  为作业指定优先级。
* Process tracking (for signaling): 提供作业进度追踪，用于信号捕获及作业信息统计。
* Scheduler: 决定作业如何调度。
* Node selection: 决定资源如何分配。
* Switch or interconnect: 交换机或互连接口插件。对于大多数系统（以太网或infiniband），这不是必需的。
* Task Affinity: 提供绑定作业的机制，并将其各个任务分配给特定的处理器。
* Network Topology：根据网络拓扑优化资源选择。用于工作分配和高级预约。  

## 基本概念

下图中展示了由Slurm守护进程所管理的实体，主要包括：
* node（节点）： 即Slurm中的计算资源。
* partition（分区）： 分区将一组节点组合成逻辑上的一个集合。
* job（作业）：特定时间为用户进行的一次资源申请或分配即可看作一个作业。
* job step（作业步）：一个作业中的一组（可能并行的）任务。 

`partition`可以被认为是作业队列，每个队列都有各种约束，如作业大小限制，作业时间限制，允许使用它的用户等。按优先级排序的作业被分配到分区内的节点上，直到该分区中的资源（节点，处理器，内存等）耗尽为止。一旦作业分配了一组节点，用户就可在这些节点上以任何配置的作业步的形式启动并行作业。即可以单个`job step`使用所有的节点，也可以多个`job step`只使用部分已分配的资源。`Slurm`负责资源的管理，你可以同时提交任意多个`job`，多个`job`将进行排队，当系统中有可用资源时`Slurm`将会为这些作业分配资源。

<img width=600px alt="Slurm entities" src="http://o6sdpimwf.bkt.clouddn.com/uploads/2017/09/entities.gif"></img>