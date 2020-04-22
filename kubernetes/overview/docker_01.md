# 浅谈容器技术-01
![](https://github.com/hanlaipeng/project-analysis/blob/master/img/docker.png)

说到容器，我们就不自觉的会想到docker，那么docker是什么呢？我们看docker的log是一条驮着很多集装箱的鲸鱼，我们可以很直观的从这个log上获取到一些信息，鲸鱼背上的集装箱（也就是容器）中装的可能是一个应用，可能是一个linux的环境，也可能是一个配置好的mysql服务等，而鲸鱼就可以驮着它到任何一个港口（也就是机器），将这个集装箱卸下， 开箱即用里面的服务。而docker就是做这么一件事的，它可以将一个服务或一个应用打包成一个容器，可以将容器搬到在不同的平台上，开箱即用该容器中的服务。而容器其实就可以理解成一个沙盒。

## 容器的隔离技术
我们现在机器上启动一个容器，在这里我启动的是busybox容器，启动指令如下：

```
$ docker run -ti busybox /bin/sh
```
我们通过-ti可以以交互的模式进入到打开的容器内部，打开容器时在容器中执行的指令为：/bin/sh，当你进入到容器的时候，在根目录下执行指令 ls，你就会发现这些目录，文件夹似曾相识，就好像在一个新的系统中是一样：

```
/ # ls
bin   dev   etc   home  proc  root  sys   tmp   usr   var
```
然后执行ps aux，你会发现1号进程就是我们在启动容器的时候设置的：/bin/sh指令

```
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    8 root      0:00 ps aux
```
然后我们在宿主机上执行 ps指令来查看/bin/sh，你会发现，宿主机给他分配的进程号是26750。

```
$ ps aux | grep /bin/sh
root     26750  0.0  0.0   1300   260 pts/0    Ss+  22:56   0:00 /bin/sh
```
后面在容器中再次创建的进程就都是容器中1号进程的子进程了，所以容器是单进程模式的。这时候你也许就会产生一个疑问，容器是如何做到隔离的呢？

说到隔离，我们就不得不说Cgroups技术和Namespace技术，简单的来说，Namespace技术隔离了“环境”，而Cgroups限制了“资源”。

正如上面所看到的关于进程的隔离就是使用了PID Namespace，同时，Linux还提供了Mount Namespace，即实现了挂载点的隔离，容器中的进程只能看到隔离的挂载点的信息；Network Namepace，即实现了网络的隔离，每一个容器都有一套独立的网络系统，容器中的进程可以看到隔离的网络设备和信息，除此之外，提供了User Namespace，IPC Namespace等。在创建容器进程的时候，其实就指定了一组所需要的Namespace的参数，已实现对整个环境的隔离，实现的效果就是容器中的进程只能看到Namespace隔离之后的容器“环境”。

这时我们就不难发现，其实容器都是共用宿主机的OS的，容器并不像虚拟机一样，有一整套完整的OS，这样容器的进程就可以和宿主机上的其他进程一样，统一由宿主机的操作系统来进行管理，所以相对于虚拟机来说，容器中的进程不会受到因为虚拟化而带来性能损失的影响，但是这也会有一个问题存在，Namespace只是做了一个限制，限制某些东西能看到，某些东西你不能看到，但是这样的隔离是不彻底的。不过，这并不影响docker技术的流行。

说完了Namespace的“环境”隔离，我们下面来说一下Cgroups技术的“资源”限制。

既然一台宿主机上可以起多个容器，那么对于容器中的一个运行中的进程来说，如果正在运行的过程中，他所需要的资源（如：CPU、Memory）让其他容器的进程抢夺完了，这显然是不合理，那么容器是如何做到资源隔离的呢？这就要说一下Linux的Cgroups技术了，它的主要作用就是限制一个进程组可以使用的资源的上限，如：CPU、Memory、磁盘等。其实通过上面的描述，你也可以看出来，容器其实就是对单个进程的一种约束和管理的技术，既然容器都是使用一个OS，那么Linux的Cgroups技术完全就可以限制容器的资源使用量的上限。

我们进入到容器的/sys/fs/cgroup目录中，你就会看到各个类型的资源限制的文件：

```
/sys/fs/cgroup # ls
blkio             cpuacct           freezer           net_cls           perf_event
cpu               cpuset            hugetlb           net_cls,net_prio  pids
cpu,cpuacct       devices           memory            net_prio          systemd
```
我们以cpu举例，进入到cpu文件夹中，你会看到有很多文件，这些文件就是对进程使用CPU资源的限制，其中，我们查看cpu.cfs\_quota_us文件中的内容：

```
/sys/fs/cgroup/cpu,cpuacct # cat cpu.cfs_quota_us
-1
```

-1就代表对CPU资源的使用量没有限制，那么也就是说这个容器中的进程理论上是可以吃满整台宿主机的CPU资源的，而cpu.cfs\_period_us文件中的内容就是在多长时间区间内，进程可以分配到这么多的CPU资源。我们可以通过docker run中的指令来控制它，我们重新启动一个容器，并限制它只能使用10%的CPU资源：

```
$ docker run -ti --rm --cpu-quota=10000 busybox /bin/sh
```
我们再次进入到容器中的/sys/fs/cgroup/cpu目录，查看cpu.cfs\_quota_us文件中的内容，就会发现已经限制了CPU的使用量

```
/sys/fs/cgroup/cpu,cpuacct # cat cpu.cfs_quota_us
10000
```

同理，我们也可以通过docker run的--memory来限制memory的使用。那么容器中的这个cgroup的文件夹是哪里来的呢？我们可以到宿主机的/sys/fs/cgroup/cpu/docker/c07a010a8cdb06b44bd7df9f7目录中看一下就知道，这是宿主机使用联合挂载的方式挂载进去的。

在了解了容器的隔离与限制之后，在下一部分，我们一起来看一下[容器的文件系统](./docker_02.md)吧。