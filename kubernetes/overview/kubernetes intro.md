### [1.Kubernetes简介](#Kubernetes简介锚点)
### [2.核心组件](#核心组件锚点)
* #### [kube-api-server](#kube-api-server锚点)
* #### [kube-controller-manager](#kube-controller-manager锚点)
* #### [kube-scheduler](#kube-scheduler锚点)
* #### [kubelet](#kubelet锚点)
* #### [kube-proxy](#kube-proxy锚点)
### [3.大致流程](#大致流程锚点)


---


<span id = "Kubernetes简介锚点"></span>
### 1.Kubernetes简介

有了Docker我们可以在宿主机上进行容器化部署，随着容器的不断增多，用户不得不对容器实施分组，以便跨所有容器提供网络、安全、资源监控等服务，即我们需要一个容器编排工具。Kubernetes(k8s)就是这样一个管理跨主机容器化应用的系统,它的功能强大，包括并不限于自动化部署、管理和扩缩容器化应用。

为了使用户能够透明的使用Docker容器，Kubernetes抽象出了一些概念。

- Node：指的是Kubernetes集群部署中的工作主机，我们可以理解为物理机或者虚拟机。集群中的Node相互连通。集群中有一台或多台Master节点，负责整个集群的调度与管理工作。其余节点受Master的管理，是具体容器化应用的承担者。
- Pod：Pod是Kubernetes集群中资源分配和调度的基本单位，一个Pod可以包含一个或多个Docker容器，这些容器会共享存储卷等资源。一般来说，一个Pod中的容器相互协作完成一个功能。
- Replication Controller(RC)：它是一种Pod控制器，虽然Pod是最小的调度单位，但我们不会直接部署及管理Pod对象，而是借助控制器。如利用RC创建、维护Pod的副本数量。如果有Pod发生异常退出，它会自动创建新的Pod来代替；如果有异常多出Pod，它会对Pod自动停止和回收。总之，RC可以保证集群中的Pod数量总是保持用户期望的数量，提高集群的容灾能力。
- Label：即标签，可以理解为分组。如对Pod及其副本打上Label，然后通过Label Selector将相同Label的Pod连接起来，这样就构成了一组Pod。
- Service：上面说到Pod可能会发生异常出现启停的状况，那么Pod的IP地址就是动态的，为了使前端客户端不受影响，Kubernetes提出Service的概念。通过Label Selector连接起来的一组Pod构成一个Service。Service的IP是固定的，用户通过Service去访问后端的Pod，而具体访问的是哪一个Pod，则由Service的访问负载均衡机制决定。


<span id = "核心组件锚点"></span>
### 2.核心组件
<span id = "kube-api-server锚点"></span>
- #### kube-api-server
   在Master节点上。api-server是Kubernetes集群的访问入口，对外提供RESTful风格的接口，是用户与集群、集群中各功能模块之间通信的枢纽。

<span id="kube-controller-manager锚点"></span>
- #### kube-controller-manager
   在Master节点上。上文提到过RC，即副本控制器，而Controller Manager是各种controller的管理者，包括RC，Node Controller,ResourceQuota Controller等等，是集群内部的管理控制中心.

<span id ="kube-scheduler锚点"></span>
- #### kube-scheduler
   在Master节点上。kube-scheduler是Kubernetes的调度器，它收集集群中所有Node的资源负载情况，然后根据某个调度算法，将新建的Pod调度到合适的Node上运行。

<span id="kubelet锚点"></span>
- #### kubelet
   Kubelet运行在所有工作Node上，负责在节点中具体创建Pod。除此之外，它还负责监听其所在工作节点的资源使用情况。kubelet会在API Server 上注册当前工作节点，定期向 Master 汇报节点资源使用情况。

<span id = "kube-proxy锚点"></span>
- #### kube-proxy
   运行在工作节点上。上文提到的Service资源对象，其背后的技术支撑就是kube-proxy。它能够按需为Service资源对象生成iptables或ipvs规则，从而捕获访问当前Service的ClusterIP的流量并将其转发到正确的后端Pod。

<span id="大致流程锚点"></span>
### 3.大致流程
①Kubernetes通过命令行使用yaml文件来创建Pod。

`kubectl apply -f XXX.yaml`

②Master的Api-Server会接到一个HTTP的POST请求。Api-Server对请求进行验证，通过后将在etcd中存储Pod对象的创建信息。

③Master的Scheduler通过Api-Server访问Etcd，从中读取要创建(即未绑定)的Pod信息以及可用的Node节点信息，然后根据某一调度策略将Pod绑定到合适的节点。Scheduler会通过Api-Server在Etcd中创建一个boundPod对象，该对象描述一个工作节点上运行的Pod信息。

④Node中的Kubelet定期通过Api-Server与Eted通信，同步boundPod的信息。发现boundPod列表有更新，且Pod绑定的节点是自己，则调用Docker api创建Pod内的containers。


