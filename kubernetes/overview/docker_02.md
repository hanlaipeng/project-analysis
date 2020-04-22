# 浅谈容器技术-02
在了解了容器的隔离技术之后，也许你会注意到在容器中有一套完整的文件系统，那么这个文件系统是怎么来的呢？我们下面来看一下容器的文件系统。

## 容器的文件系统
我们都知道，容器是具有一致性的，也就是说，无论是在本地还是云端的任何一个地方，只要你run起来同一个容器镜像，那么这个容器中的运行环境就会重现出来，这个运行环境是一致的。但是如果我们需要升级这个镜像的时候，我们会发现无论我们使用dockerfile还是docker commit，都是基于现有的镜像，经过改动升级成为一个新的镜像的，并没有影响到基础镜像中的环境，没错，这个就是docker的分层概念（layer），docker将每一个增量（每一步操作）以一层的形式保存了下来，这样用户都只需要去维护这些增量部分就可以了，不用关心base镜像。

下面我们来一起构建一个docker镜像test:v1，你就明白了，dockerfile如下：

```
FROM centos:latest
RUN yum install -y vim
ADD start.sh /
```
该镜像的基础镜像是centos:latest（注意：请不要这样引用基础镜像，latest一般会持续更新，所以一旦latest的镜像更新的话，那么你重新构建镜像的时候就会发现，你的镜像的基础环境也更新了），这里面安装vim指令，添加了一个文件start.sh，我们通过docker的inspect指令查看该镜像的详情，我们可以看到它总共有三层：

```
$ docker image inspect test:v1
......
"RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:0683de2821778aa9546bf3d3e6944df779daba1582631b7ea3517bb36f9e4007",
                "sha256:37a378709ad0d87f06676360954fdd2fff510b6f0f0701c6936ea98303608979",
                "sha256:d7b635db2ba4b3ef7db44ccc2ca2325771bc289ffb84c802ddddb7a00c374b00"
            ]
        }
......
```
由于容器是使用宿主机的OS，所以这些层中其实是保存了所需要的运行的文件、配置和目录，而这些操作系统所需的文件目录配置称为：rootfs。那么我们就可以想到，在容器启动的时候，还需要把这些文件联合挂载到容器中，为容器进程提供隔离后的执行环境的文件系统，既然是隔离，那么我们自然而然的就会想到Namespace技术的mount，由于是分多个层，那么docker采用的是联合挂载技术，我的环境中用的是overlay2，和AUFS类似，关于overlay2的介绍与配置可以去docker的官方文档中查看[https://docs.docker.com/storage/storagedriver/overlayfs-driver/](https://docs.docker.com/storage/storagedriver/overlayfs-driver/) ，下面我们看一下docker是如何联合挂载的。

我们依然使用inspect指令查看test:v1的详情：

```
$ docker image inspect test:v1
......
"GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/4b21e5d5113a0e7948148b66fbd9376035854eef3180e8c3abc6fd8be1d5137a/diff:/var/lib/docker/overlay2/7b261f874234811d8ac41c81e63c9ea57a8e5ee1db4d9f2fa47bd0c206094694/diff",
                "MergedDir": "/var/lib/docker/overlay2/7cea560f878dc632a48d46f45e0add70cf034ae4c8084bcb1b9f9e6207fde805/merged",
                "UpperDir": "/var/lib/docker/overlay2/7cea560f878dc632a48d46f45e0add70cf034ae4c8084bcb1b9f9e6207fde805/diff",
                "WorkDir": "/var/lib/docker/overlay2/7cea560f878dc632a48d46f45e0add70cf034ae4c8084bcb1b9f9e6207fde805/work"
            },
            "Name": "overlay2"
        }
......
```
我们先看LowDir，
