# Docker 

>Docker是一个开源的引擎，可以轻松的为任何应用创建一个轻量级的、可移植的、自给自足的容器。开发者在笔记本上编译测试通过的容器可以批量地在生产环境中部署，包括VMs（虚拟机）、 bare metal、OpenStack 集群和其他的基础应用平台。

## 1. Docker 三要素
  
 docker 三要素：镜像，容器，仓库

 1.镜像（Image）

    Docker镜像（Image）就是一个只读的模板，镜像可以用来创建Docker容器，一个镜像可以创建很多容器。
 2.容器（Container）

    容器是使用镜像创建的运行实例。
    docker利用容器独立运行一个或一组应用。
    它可以被启动，停止，删除，每个容器都是相互隔离，保证安全的平台。
    可以把容器看作一个建议的Linux系统

 > 镜像可以看作面向对象中的类，容器可以看作类的实例，每个实例都是独立的，拥有类的所有属性和方法，一个类可以实例化多个实例。
 
 3.仓库（Repository）

    集中存放镜像的地方，类比 GitHub，有公开仓库和私有仓库，
    最大的公开仓库是 Docker Hub,国内有阿里云，网易云等公开仓库。

## 2. Docker的安装

### 2.1 安装 Docker CE

	# step 1: 安装必要的一些系统工具
	sudo apt-get update
	sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
	
	# step 2: 安装GPG证书
	curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
	
	# step 3: 检查验证密钥是否添加成功
	sudo apt-key fingerprint 0EBFCD88
	# 如果成功，则显示类似下面：
	# pub   rsa4096 2017-02-22 [SCEA]
	#      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
	# uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
	# sub   rsa4096 2017-02-22 [S]
	
	# Step 4: 写入软件源信息
	sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
	
	# Step 4: 更新并安装 Docker-CE
	sudo apt-get -y update
	sudo apt-get -y install docker-ce

### 2.2 阿里云镜像加速

   先去申请一个加速器地址 链接
   然后在下面的 “镜像中心 -> 镜像加速器” 按指示添加，Ubuntu的是：

	sudo mkdir -p /etc/docker
	sudo tee /etc/docker/daemon.json <<-'EOF'
	{
	  "registry-mirrors": ["https://d7gtkri7.mirror.aliyuncs.com"]
	}
	EOF
	sudo systemctl daemon-reload
	sudo systemctl restart docker

  最后使用docker info检查一下有没有生效

## 3. HelloWorld

  执行 docker run hello-world

	root:~# docker run hello-world
	
	Unable to find image 'hello-world:latest' locally
	latest: Pulling from library/hello-world
	1b930d010525: Pull complete 
	Digest: sha256:9572f7cdcee8591948c2963463447a53466950b3fc15a247fcad1917ca215a2f
	Status: Downloaded newer image for hello-world:latest
	
	Hello from Docker!
	This message shows that your installation appears to be working correctly.
	
	......

 hello-world 是docker 官方为了方便测试docker是否安装成功提供的镜像，运行这个镜像，首先会在本地找，没找到就回去仓库下载

 hello-world:latest 后面的latest 是标签名，不写默认的，需要下载其他版本的就该在冒号后面的。

## 4. 基本命令

 docker version：                             查看版本号
 
 docker info:                                 查看docker相关信息

 docker help:                                 查看常用命令帮助

 docker images:                               列出本地镜像模板

 docker search: 镜像名

 docker pull: 拉取镜像

 docker rmi: 删除镜像
 
  docker build -f dockerFile路径 -t [命名空间/]镜像名 ./   根据DockerFile构建镜像
  
  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]     启动容器
  
  docker stop [容器ID]   停止容器
  
  docker logs [-f] [-t] [--tail] [容器ID]      查看日志    

 docker exec -ti [容器ID]       进入容器
 
 docker run -it -v /宿主机绝对路径目录:/容器内目录 [镜像名]   挂载文件目录到容器中        




