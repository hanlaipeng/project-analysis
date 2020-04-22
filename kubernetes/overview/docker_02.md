# 浅谈容器技术-02
在了解了容器的隔离技术之后，也许你会注意到在容器中有一套完整的文件系统，那么这个文件系统是怎么来的呢？我们下面来看一下容器的文件系统。

## 容器的文件系统
我们都知道，容器是具有一致性的，也就是说，无论是在本地还是云端的任何一个地方，只要你run起来同一个容器镜像，那么这个容器中的运行环境就会重现出来，这个运行环境是一致的。但是如果我们需要升级这个镜像的时候，我们会发现无论我们使用dockerfile还是docker commit，都是基于现有的镜像，经过改动升级成为一个新的镜像的，并没有影响到基础镜像中的环境，没错，这个就是docker的分层概念（layer），docker将每一个增量（每一步操作）以一层的形式保存了下来，这样用户都只需要去维护这些增量部分就可以了，不用关心base镜像。

下面我们来一起构建一个docker镜像test:v1，你就明白了，dockerfile如下：

```
FROM centos:latest
ADD start.sh /
```
该镜像的基础镜像是centos:latest（注意：请不要这样引用基础镜像，latest一般会持续更新，所以一旦latest的镜像更新的话，那么你重新构建镜像的时候就会发现，你的镜像的基础环境也更新了），这里面添加了一个文件start.sh，我们通过docker的inspect指令查看该镜像的详情，我们可以看到它总共有两层：

```
$ docker image inspect test:v1
......
"RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:0683de2821778aa9546bf3d3e6944df779daba1582631b7ea3517bb36f9e4007",
                "sha256:d7b635db2ba4b3ef7db44ccc2ca2325771bc289ffb84c802ddddb7a00c374b00"
            ]        
}
......
```
由于容器是使用宿主机的OS，所以这些层中其实是保存了所需要的运行的文件、配置和目录，而这些操作系统所需的文件目录配置称为：rootfs。那么我们就可以想到，在容器启动的时候，还需要把这些文件联合挂载到容器中，为容器进程提供隔离后的执行环境的文件系统，既然是隔离，那么我们自然而然的就会想到Namespace技术的mount，由于是分多个层，那么docker采用的是联合挂载技术，我的环境中用的是overlay2，和AUFS类似，关于overlay2的介绍与配置可以去docker的官方文档中查看[https://docs.docker.com/storage/storagedriver/overlayfs-driver/](https://docs.docker.com/storage/storagedriver/overlayfs-driver/) ，下面我们看一下docker是如何联合挂载的。

我们将test:v1镜像run起来，通过mount指令来查看容器文件系统的mount情况：

```
$ mount | grep overlay
overlay on /var/lib/docker/overlay2/59f45e3f9d1034970d54701de694e24de070e3c80f2a4b70bc842330c4b49137/merged type overlay 
(rw,relatime,lowerdir=/var/lib/docker/overlay2/l/3XYKSIO2NPISQGROGKULXGGSVZ:/var/lib/docker/overlay2/l/ABDMAE6YGOAW43SNPZISP2ML6Y:/var/lib/docker/overlay2/l/WGT7Q6M3U3FCGYPOKXMWQAC4ID,
upperdir=/var/lib/docker/overlay2/59f45e3f9d1034970d54701de694e24de070e3c80f2a4b70bc842330c4b49137/diff,
workdir=/var/lib/docker/overlay2/59f45e3f9d1034970d54701de694e24de070e3c80f2a4b70bc842330c4b49137/work)
```
我们可以很清晰的看到容器的mount的信息，这个mount信息中也隐藏了容器的层级信息，如下图：

下面我们来分析一下这些mount的信息，一起揭露容器的层级信息：

### lowerdir：
从这个名称我们就可以看出，它处在最低层，我们也称之为只读层，为什么叫做只读层呢？因为这些层就是我们构建镜像的层，我们来看一下：

* 只读层

我们查看目录/var/lib/docker/overlay2/l/WGT7Q6M3U3FCGYPOKXMWQAC4ID的信息，发现它是一个链接，我们进入到这个链接所指向的目录的上一层中查看：

```
$ ls | grep WGT7Q6M3U3FCGYPOKXMWQAC4ID
WGT7Q6M3U3FCGYPOKXMWQAC4ID -> ../7b261f874234811d8ac41c81e63c9ea57a8e5ee1db4d9f2fa47bd0c206094694/diff

$ ls
diff  link

$ ls diff
bin  etc   lib    lost+found  mnt  proc  run   srv  tmp  var
dev  home  lib64  media       opt  root  sbin  sys  usr

$ cat link
WGT7Q6M3U3FCGYPOKXMWQAC4ID
```

很明显我们可以看出，其中diff文件中的文件就是我们的基础镜像Centos的信息，包含了操作系统所需要的基本文件目录与配置，link中保存的就是diff的链接，加上这个链接是为了防止mount的时候路径过长。

我们查看目录/var/lib/docker/overlay2/l/ABDMAE6YGOAW43SNPZISP2ML6Y的信息，并进入到这个链接所指向的目录的上一层中查看：

```
$ ls
diff  link  lower  work

$ ls diff
start.sh

$ cat link
ABDMAE6YGOAW43SNPZISP2ML6Y

$ cat lower
l/WGT7Q6M3U3FCGYPOKXMWQAC4ID
```

我们可以看到，diff中就是我们新添加的层中的信息start.sh文件，其中lower文件中记录了只读层的链接号，就像链表的指针一样，如果我们的dockerfile中运行更多的指令，这里就会多一些这样的层。说到这，你会发现还多一个层，没错，这个层就是只有容器运行起来的时候才有的init层。

* init层

我们查看目录/var/lib/docker/overlay2/l/3XYKSIO2NPISQGROGKULXGGSVZ的信息，并进入到这个链接所指向的目录的上一层中查看，你会发现目录的结尾是"-init"：

```
$ ls
diff  link  lower  work

$ ls diff
dev  etc

$ cat link
3XYKSIO2NPISQGROGKULXGGSVZ

$ cat lower
l/ABDMAE6YGOAW43SNPZISP2ML6Y:l/WGT7Q6M3U3FCGYPOKXMWQAC4ID
```
这是一个特殊的层，是docker单独生成的内部层，它里面的主要存的配置文件为：hostname、hosts、resolv.conf等配置文件，这个层的存在是因为用户在启动容器的时候一般需要写入一些临时的配置值，但是我们只希望这个修改的配置只对当前的容器有效，不希望可以通过commit提交到一个新的镜像中，所以，在启动容器的时候init层就出现了。并且我们可以在lower中可以看到它指向了只读层。

### upperdir：

* 可读写层

从这个名称就可以看出来，它是最上层的，我们称之为可读写层，这一层主要记录了我们在容器中的一切修改，等到用户使用commit提交的时候，就将这一层和下面的只读层一起打包成为一个新的镜像，我们进入到目录/var/lib/docker/overlay2/59f45e3f9d1034970d54701de694e24de070e3c80f2a4b70bc842330c4b49137中查看：

```
$ ls
diff  link  lower  merged  work

$ cat lower
l/3XYKSIO2NPISQGROGKULXGGSVZ:l/ABDMAE6YGOAW43SNPZISP2ML6Y:l/WGT7Q6M3U3FCGYPOKXMWQAC4ID
```
默认diff中为空，如果我们在容器中新建一个文件夹aaa，那么diff中就会出现aaa这个文件夹，不出意料，我们发现它的lower指向的是只读层和init层。

### merged 和 work：

我们在upperdir的上一级目录中发现了和diff在同一级的目录，merged 和 work，那么它们是干嘛的呢？
merged：是联合挂载的时候，lowerdir和upperdir的合并结果，合并过程如下图：

![](https://github.com/hanlaipeng/project-analysis/blob/master/img/docker-merge.png)

`其中上层如果和下层重复，则上层覆盖下层的。但是，这里的覆盖不是替换，下面的层中的内容还是会存在的，所以这里就有了一个瘦身镜像的注意事项，你在构建镜像的时候，除了选择比较小的基础镜像，还一定要注意不要出现下层安装，上层删除的情况，这个不会出现1-1=0的情况，可以考虑使用阶段构建。`
work：这个目录是一个空的目录，它主要是overlayFS在内部使用的一个目录。

我们都说容器单独存在是没有意义的，只有容器的编排才有意义，那么下面我们就一起进入kubernetes项目中，先了解一下它的最小运行单元pod吧。
