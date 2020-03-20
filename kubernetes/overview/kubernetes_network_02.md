## kubernetes集群的通信
我们在[上一篇](./kubernetes_network_01.md)主要介绍了docker同宿主机，以及访问外部主机的的通信方式与原理，并且在最后也提出了容器跨主机访问的时候所需要用的网络插件Flannel和Calico，那么本篇我们就主要聊一下这个网络插件的工作原理，以及kubernetes集群的通信。

为什么要跨过介绍容器跨主机通信而直接到kubernetes集群的通信呢，这是因为kubernetes主要的功能就是容器的编排，虽然kubernetes的最小单元是pod，但是pod的核心依然是容器，所以，kubernetes集群pod之间的通信原理其实和容器通信的原理是一致的，所以我们直接介绍kubernetes集群的通信。

在上一篇中我们提到，docker的每个宿主机上都会起一个类似网桥功能的docker0，用于将容器中的请求“拿出”到主机中，然后再根据IP表进行下一步的转发，kubernetes同样也实现了这样的一个“网桥”就是cni0，如果说docker0网桥上挂的是一个一个容器，那么cni0网桥上挂的就是一个一个pod，而挂起的方式同样是通过Veth Pair设备实现的。那为什么kubernetes不用docker的docker0而要自己实现一个cni0呢？这就和pod的网络配置有关了。

```
    pod的网络配置：
    由于容器的本质是进程，但是一般情况下任务是由多个进程共同执行的，所以需要pod就将多个相关容器组成一个执行单元，而这个执
行单元需要共用同一套网络栈，这个时候cni0就派上用场了。细心的你会发现当你起一个pod的时候，在宿主机执行docker ps 的时候会
发现会先起一个pause-amd64容器，它就会先调用CNI插件，为这个pod配置网络资源，这个容器很小很小基本上是不占用资源的，所以它
不会影响pod中其他容器的运行，然后启动定义的一组容器，然后这一组容器共享这一套网络资源。
```

我们知道docker在跨主机通信的时候为了防止IP出现冲突，有了子网的概念，而kubernetes中同样也有子网的概念，可以看出主机A和B的pod是在不同的子网中，而pod的IP也是严格按照这个子网来进行分配的。

```
主机A
cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 
       inet 10.244.1.1/24 
                   
主机B
cni0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
       inet 10.244.2.1/24  
```
有了当pod创建成功之后就可以开始发起请求，在这里我们先从Flannel网络插件说起。
## Flannel
Flannel项目是CoreOS公司主推的一个容器跨主访问的容器网络方案，真正实现这个网络方案的是Flannel的三种后端实现：VXLAN、host-gw、UDP，其中UDP是最早支持的一种模式，但是由于它在发送请求的过程中要经历三次用户态与内核态的转换导致性能比较差，已经弃用了，所以在这里我们不多做介绍，我们先来介绍一下VXLAN实现模式。

我们拿主机A举例，我们在安装完Flannel的时候，我们可以发现机器上会多出一个flannel.1网卡，而它的IP就是主机A子网中的IP

```
  $ ifconfig
  flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether d2:2b:a1:6b:db:a6 brd ff:ff:ff:ff:ff:ff
    inet 10.244.1.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::d02b:a1ff:fe6b:dba6/64 scope link
       valid_lft forever preferred_lft forever
```
我们查看路由表就会发现主机A的路由表中也添加上一条新的路由规则

```
 $ route
10.244.0.0      10.244.0.0      255.255.255.0   UG    0      0        0 flannel.1
10.244.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.2.0      10.244.2.0      255.255.255.0   UG    0      0        0 flannel.1
```
由此可见，当主机A上的pod向主机B的pod发送请求的时候，由于主机B中pod的子网是10.244.2.0/24，所以在这个请求通过Veth Pair出现在主机A中的时候，根据路由表则可以匹配到第三条，走flannel.1网卡发送出去。

```
	细心的用户可以发现，当集群中有一台机器加入的时候，该子网的信息会同步到集群中的所有的route表中，这是Flannel的一个主
要功能。所以，在主机A中才可以看到有关主机B的子网信息。
```
VXLAN，即 Virtual Extensible LAN（虚拟可扩展局域网），是 Linux 内核本身就支持的一种网络虚似化技术。所以说，VXLAN 可以完全在内核态实现上述封装和解封装的工作



	
