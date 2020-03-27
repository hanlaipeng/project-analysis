### [1.什么是Kubernetes-scheduler?](#什么是Kubernetes-scheduler?锚点)
### [2.Kubernetes调度器的工作流程](#Kubernetes调度器的工作流程锚点)
### [3.Kubernetes调度器的实现细节](#Kubernetes调度器的实现细节锚点)
### [4.Scheduler中Pod优先级和抢占策略](#Scheduler中Pod优先级和抢占策略锚点)
---

<span id="什么是Kubernetes-scheduler?锚点"></span>
### 1.什么是Kubernetes-scheduler?
当集群中Master节点的API-Server将创建Pod的请求存储至Etcd后，有一个关键的步骤，即通过调度器程序(Kubernetes Scheduler)从当前集群中选择一个可用的最佳节点来接受并运行它，从而实现基于资源可用性将Pod资源公平的分配于集群节点之上。

我们在集群中通常是混合部署应用程序，这些应用可能是高CPU消耗的，高IO的，内存敏感的，或者GPU消耗的等等，由此导致调度问题复杂化，资源调度设计者需要考虑更多的限制条件和调度策略。

<span id="Kubernetes调度器的工作流程锚点"></span>
### 2.Kubernetes调度器的工作流程
在Kubernetes_intro中我们简单介绍了Kubernetes创建一个Pod的流程，而Scheduler调度器做为Kubernetes三大核心组件之一， 承载着整个集群资源的调度功能，所以我们对Scheduler的运作做一个具体的说明。

![Scheduler流程图](https://github.com/hanlaipeng/project-analysis/raw/master/img/Scheduler/scheduler_flow.png)

- 调度器内部维护一个调度的pods队列podQueue， 并监听APIServer。
- 当我们创建Pod时，首先通过APIServer 往ETCD写入pod元数据。
- 调度器监听pods状态，当有新增pod时，将pod加入到podQueue中。
- 调度器中的主进程，会不断的从podQueue取出的pod，并将pod进入调度分配节点环节。
- 分配节点成功， 调用apiServer的binding pod 接口， 将pod.Spec.NodeName设置为所分配的那个节点。
- 节点上的kubelet同样监听ApiServer，如果发现有新的pod被调度到所在节点，调用本地的dockerDaemon 运行容器。
- 假如调度器尝试调度Pod不成功，如果开启了优先级和抢占功能，会尝试做一次抢占，将节点中优先级较低的pod删掉，并将待调度的pod调度到节点上。 如果未开启，或者抢占失败，会记录日志，并将pod加入podQueue队尾，即等待。


<span id="Kubernetes调度器的实现细节锚点"></span>
### 3.Kubernetes调度器的实现细节

- 是共享状态的资源调度模型
- 采用静态资源调度策略

我们将Kubernetes集群调度器简单地看作是一个输入/输出的黑盒，那么输入就是待调度Pod和可用的工作节点列表，输出则是根据调度策略从可用的工作节点列表中选举出来的一个工作节点。

![Scheduler调度策略](https://github.com/hanlaipeng/project-analysis/raw/master/img/Scheduler/scheduler_strategy.png)


调度策略由三部分组成，分别是：预选、优选和选定。

①预选过程中检查所有可用节点，筛选出符合 Pod要求的节点。这些节点成为下一流程的候选节点。

②在预选流程筛选出来的节点中进一步进行筛选。使用优选策略分别计算出每一个可用节点的得分，得分最高的节点则为最适合被调度的节点。

③如果优选调度策略计算出来的得分结果中，同时有两个或多个节点为最高分节点，那么就利用随机算法从这些节点中随机选择一个进行调度。

由于Kubernetes集群调度器提供了一个可插拔的算法框架，开发者能够很方便地往调度器添加各种自定义的调度算法，甚至添加自定义的调度器。本节我们主要研究已经集成的预选和优选策略。

- 预选策略
    - PodFitsPorts：端口是否冲突。检查其值指定的端口是否已被节点上的其他容器或服务占用
    - PodFitsResources：资源是否够用
    - NoDiskConflit：挂载的磁盘是否冲突，检查Pod对象请求的存储卷在此节点是否可用

- 优选策略
    - LeastRequestedPriority：该策略从可用节点当中选择出目前资源消耗最少的节点，资源消耗越少的节点得分越高。
    - BalancedResourceAllocation：该策略从可用节点当中选择各项资源如 CPU、内存使用率最均衡的节点，资源越均衡的节点得分越高
    - SelectorSpreadPriority：首先查找与当前 Pod 对象匹配的 Service、ReplicationController、 ReplicaSet ( RS ）和 StatefulSet，而后查找与这些选择器匹配的现存 Pod 对象及其所在的节 点，则运行此类 Pod 对象越少的节点得分将越高。 简单来说，如其名称所示，此优选函数 会尽量将同一标签选择器匹配到的 Pod 资源分散到不同的节点上运行。

<span id="Scheduler中Pod优先级和抢占策略锚点"></span>
### 4.Scheduler中Pod优先级和抢占策略
Kubernetes自1.8起开始支持Pod资源的的优先级机制，它用于表现一个 Pod对象相对于其他Pod对象的重要程度。一个Pod对象无法被调度时，调度器会尝试抢占（驱逐）较低优先级的Pod对象，以便可以调度当前Pod。 

另外，在 Kubernetes1.9和更高版本中，优先级还会影响节点上Pod的调度顺序和驱逐次序，即高优先级的Pod会优先被调度，或者在资源不足的情况下牺牲低优先级的Pod。

我们知道Scheduler维护一个待调度队列，当Pod拥有优先级这一属性之后，高优先级的Pod就会比低优先级的提前出队。

而当一个高优先级的Pod调度失败的时候，调度器的抢占能力就会被触发。这时，调度器就会试图从当前集群里寻找一个节点，使得当这个节点上的一个或者多个低优先级Pod被删除后，待调度的高优先级Pod就可以被调度到这个节点上。这个过程，就是“抢占”。

在抢占过程中，调度器会将抢占者Pod的spec.nominatedNodeName设置为要抢占的Node名，同时该Node上被Kill的Pod有优雅退出的时间：如果在该过程中有别的可调度节点，或者有更高优先级的Pod也抢占该Node,那么抢占者的nodeName最终不一定会是nominatedNmae,这也是设置nominatedName的原因。









  
 



