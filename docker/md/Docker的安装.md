# Docker的安装

我们介绍完了docker的基本概念，那么现在让我们着手安装一个docker吧。

## 安装docker

首先我们需要打开官方文档来查找在各种系统上安装docker的文档。  [docker安装文档地址](https://docs.docker.com/)

我们选择社区版的安装文档。

p1.png

我们可以看到 docker支持云服务器安装，linux系统安装，以及MacOS和 Windows安装。

由于我使用的是linux的centos系统，所以我这里只介绍centos上的安装，其实只要按照文档上的步骤来，任何系统下的安装都是可以成功的。



## CentOS安装docker

p2.png

我们打开对应的安装文档页面，按照文档的步骤来安装一下，我使用的是阿里云的轻量服务器，因此是可以联网访问的所以安装非常简单。

1.第一步如果系统内有老版本的docker，我们需要先删除之前的docker以及相关依赖。

```shell
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

我们执行这段命令

p3.png



2.安装社区版docker

2.1 安装所需要的包。

```shell
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```

p4.png

p5.png

最后出现complete表示安装完成。

2.2 设置稳定的存储库

```shell
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

p6.png



3.安装最新的社区版docker

```shell
sudo yum install docker-ce docker-ce-cli containerd.io
```

p7.png

p8.png

p9.png

4.启动docker服务

```shell
sudo systemctl start docker
```

p10.png

5.运行一个 `hello-world`镜像，来检测社区版的docker是否安装成功

```shell
sudo docker run hello-world
```

p11.png



出现了`Hello from Docker!`表示安装成功。



## 总结

其他系统的安装只要按着文档的步骤来，相信不会有问题，这里最好是自己弄一个虚拟机，或者买一个云服务器，只要单纯的centos系统做实践就好。