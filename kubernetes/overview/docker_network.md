# 浅谈容器网络
当我们在开发机上run起来一个container的时候，细心的同学就会发现容器内的IP地址是一个“特殊”的IP地址，并且如果宿主机在联网的情况下我们发现在容器里是可以访问到外面的网络，并且在同一台机器上的容器又是可以相互ping通的，甚至在“某些条件下”，不同宿主机的容器也是可以相互通信的。那么它是如何做到的呢？这个“某些条件”又是什么条件呢？首先我们先从同宿主机容器与容器之间的通信说起。
## 同宿主机上容器的通信
我们知道容器是通过Cgroup和namespace进行资源隔离从而可以像一台独立的机器一样运行，那么两台机器如果想进行通信，那么我们可以通过交换机把他们连在同一个局域网中，那么如果同一宿主机上的两个容器想通信呢？那我们就需要给他们之间搭上一个“桥”，而docker通过在宿主机上创建docker0这么一个网桥，将容器都“连接”在这个网桥上，实现了容器之间的通信。您可以通过下面指令来进行查看：

```
$ ip addr show
 3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:8d:f2:68:07 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:8dff:fef2:6807/64 scope link
       valid_lft forever preferred_lft forever
```
有了docker0这个网桥之后，那么容器又是如何连接在这个网桥上的呢？这就不得不说Veth Pair虚拟设备了，这个虚拟设备时是成对出现的，一个在容器中，另一个连接在docker0上，成为docker0的从设备，有了Veth Pair，容器就可以连接在docker0网桥上了。我们在宿主机上run一个centos的镜像并进入到容器中。

```
$ docker run -d -ti --rm --name centos-a centos
$ docker exec -ti centos-a /bin/bash
```
进入到容器中我们可以查看到它的网络设备：

```
[root@296ea2530c48 /]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.3  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:03  txqueuelen 0  (Ethernet)
        RX packets 41712  bytes 85568090 (81.6 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 34316  bytes 2491555 (2.3 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        
[root@296ea2530c48 /]# cat /sys/class/net/eth0/iflink
1198
```
我们可以看到容器的网络设备中有一个eth0的网卡，通过这个网卡的信息，我们可以看到这个容器的IP地址为172.17.0.3，这个就是Veth Pair虚拟设备在容器中的一端，与之相对应的另一端的虚拟设备我们可以在容器中的/sys/class/net/eth0/iflink文件中查看的到为1198，在宿主机上的网络设备中查看到，它的名字就是veth6935999的一个虚拟设备：

```
$ ifconfig
1198: veth6935999@if1197: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether d2:07:e8:d3:82:15 brd ff:ff:ff:ff:ff:ff link-netnsid 30
    inet6 fe80::d007:e8ff:fed3:8215/64 scope link
       valid_lft forever preferred_lft forever
```
我们在宿主机上通过网桥查看工具brctl就可以查看到veth6935999这个虚拟设备是插在docker0上的：

```
$ brctl show
bridge name	   bridge id	        STP enabled	   interfaces
docker0		   8000.02428df26807	no		       veth6935999
```
有了docker0网桥和Veth Pair虚拟设备，同宿主机的容器就可以愉快的互相通信了。下面我们看一下它们的通信细节。

由于docker0是以网桥的身份存在，那么它就是工作在数据链路层的一个虚拟网络设备，既然是工作在数据链路层，那么它就是根据MAC地址来进行包传递的，我们同时在同一台宿主机上再起一个centos的镜像，centos-b，我们进入到centos-b中可以查看到它的网络设备：

```
[root@934aafff5f1b /]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.5  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:05  txqueuelen 0  (Ethernet)
        RX packets 5916  bytes 14482340 (13.8 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4568  bytes 312719 (305.3 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        
[root@934aafff5f1b /]# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         gateway         0.0.0.0         UG    0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 eth0
```
可以看到该网络设备的IP地址为：172.17.0.5，通过route可以查看到它的route表中第二条，如果要访问的目的地址和172.17.0.0/16匹配的话，默认就会走eth0这个网卡发送出去，我们看它的网关为0.0.0.0，说明它是直连网络，可以直接发送的。这样我们就不难想象的到，如果容器centos-b要访问容器centos-a的话，根据centos-a的IP地址：172.17.0.3 这个请求地址为：172.17.0.5 目的地址为172.17.0.3的请求包就可以直接匹配到172.17.0.0/16，也就是路由表中的第二条，通过eth0网卡发送到docker0网桥上。

但是docker0是网桥是工作在数据链路层的，所以这个请求包需要拿着目的地址IP获取目的地址MAC，封装一层这个请求包，才能发送出去，这个时候就用到了地址解析协议ARP，这个协议实现了从IP地址到MAC地址的映射，即询问目标IP对应的MAC地址。由于所有的镜像都是通过Veth Pair虚拟设备挂载docker0之上，那么docker0就可以广播收到ARP的请求，然后，centos-a容器就会收到这个请求，然后将自己的MAC地址告知centos-b。

容器在拿到目的地址的MAC地址之后，就可以将目的MAC地址封装在请求包上，通过通过容器中eth0网卡和挂载docker0上的虚拟网卡发送到docker0网桥上，docker0则将请求包发送到指定的centos-a的Veth Pair虚拟设备上，通过这个虚拟设备请求流入容器centos-a中，从而实现了同一宿主机容器之间的通信。

通过在宿主机上执行tcpdump监控docker0抓包，在容器centos-a中pingcentos-b就可以看到这个过程：

```
$ tcpdump -i docker0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on docker0, link-type EN10MB (Ethernet), capture size 262144 bytes
23:08:15.206240 ARP, Request who-has 172.17.0.5 tell 172.17.0.3, length 28
23:08:15.206679 ARP, Reply 172.17.0.5 is-at 02:42:ac:11:00:05 (oui Unknown), length 28
23:08:15.206955 IP 172.17.0.3 > 172.17.0.5: ICMP echo request, id 370, seq 1, length 64
23:08:15.207341 IP 172.17.0.5 > 172.17.0.3: ICMP echo reply, id 370, seq 1, length 64
23:08:16.207823 IP 172.17.0.3 > 172.17.0.5: ICMP echo request, id 370, seq 2, length 64
23:08:16.207905 IP 172.17.0.5 > 172.17.0.3: ICMP echo reply, id 370, seq 2, length 64
23:08:17.207748 IP 172.17.0.3 > 172.17.0.5: ICMP echo request, id 370, seq 3, length 64
23:08:17.207840 IP 172.17.0.5 > 172.17.0.3: ICMP echo reply, id 370, seq 3, length 64
23:08:18.207693 IP 172.17.0.3 > 172.17.0.5: ICMP echo request, id 370, seq 4, length 64
23:08:18.207761 IP 172.17.0.5 > 172.17.0.3: ICMP echo reply, id 370, seq 4, length 64
23:08:19.207687 IP 172.17.0.3 > 172.17.0.5: ICMP echo request, id 370, seq 5, length 64
23:08:19.207746 IP 172.17.0.5 > 172.17.0.3: ICMP echo reply, id 370, seq 5, length 64
```
## 容器访问另一台宿主机
如果宿主机是联网的，那么在容器中你可以发现是可以ping通www.baidu.com这个网址的，那么请求是如何发送出去的呢？其实在了解了同宿主机容器之间的通信这个就非常好理解了。我们可以在宿主机上查看到它有一个eth0网卡，通过route指令就可以看到宿主机的route表：

```
$ route
```
容器中的请求会先发送到docker0上，然后通过路由表匹配，就可以通过eth0网卡将请求发送出去。具体的流程您也可以通过tcpdump指令抓包获取到请求的流程。在了解了同宿主机容器之间的通信原理与容器与另一台宿主机通信的原理之后，我们后面会着重介绍一下不同宿主机容器之间是如何通信的。

## 不同主机的容器通信
当主机A的容器访问三层联通的主机B的时候，我们可以发现是可以访问到的，但是如果我们想要从主机A的容器中访问主机B中的容器的时候，该怎么访问的呢？这个时候就不得不说docker中的一个重要的概念，就是子网，既然要两个不同主机的容器进行通信，那么就需要有不同的IP地址，不能目的IP和源IP是一样的，这样在发送的时候网络就会蒙圈了，所以一个集群中，每台机器的docker0都有一个和其他机器不同的子网，

```
  比如：主机A的docker的子网为：172.17.0.0，那么主机A中的容器所划分的IP地址为：172.17.0.xx 
```

那么为了保证集群中容器的IP的唯一性，那么主机B的子网就必须和A的不同

```
  比如：主机B的docker的子网为：172.17.1.0，那么主机B中的容器所划分的IP地址为：172.17.1.xx
```

那么这就为容器之间的通信创造了可能性。

但是有了子网的概念还是远远不够的，那么在发送请求的时候，我们怎么根据这个容器目的IP地址获取目的主机IP呢？可以想象到，如果我们有一个进程或者一个设备帮我们找到目的容器IP所在的主机IP，那么我们就可以实现将这个请求包封装之后发往其他的主机，然后目的主机拆包之后，通过该进程或设备识别出这个请求的目的地址是该主机上的容器，就可以将这个请求发给docker0，通过docker0将该请求发到目的容器中，以实现不同主机的容器通信。那么这个进程或设备就是我们所常说的网络插件，下篇文章主要介绍一下常用的两种网络插件：Flannel和Calico