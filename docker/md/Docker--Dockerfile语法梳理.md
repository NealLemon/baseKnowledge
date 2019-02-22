# Dockerfile语法梳理

在上一篇 [使用Docker部署第一个Springboot应用](https://www.jianshu.com/p/d998dc9d9685) 中我们使用了`Dockerfile`来创建镜像，接下来让我们来进一步的了解一下`Dockerfile`的语法。

## Dockerfile语法

我们先来看一下上篇的 `Dockerfile`

```shell
#获取base image
FROM adoptopenjdk/openjdk8:latest 
#类似于执行 linux指令
RUN mkdir /opt/app  
#类似于linux copy指令
COPY demo-docker.jar /opt/app/       
#对外端口
EXPOSE 8080
#执行命令 java -jar /opt/app/demo-docker.jar
CMD ["java", "-jar", "/opt/app/demo-docker.jar"]
```

我们可以看到 简单的几行命令就可以使用docker自动的部署我们自己的项目。接下来让我们来仔细的了解一下每个命令。

#### FROM

一般是Dockfile开头的语法，他的作用是指定我们所新创建的docker image的base image,比如

```shell
 FROM scratch             # 表示我们从头去创建image 不依赖于base image
 FROM adoptopenjdk/openjdk8:latest  # 表示从某个base image之上创建image
```

我们可能好奇，`adoptopenjdk/openjdk8:latest`这段代码的含义是什么，其实这个是我调用了 docker hub上已经有人发布的镜像，不需要自己去配置JDK的环境。类似于github,我们可以在牛人已经实现的代码仓库中下载所需要的代码。在这里我们顺便介绍一下 docker hub的简单使用，因为是新手，所以不会去创建docker hub的镜像，我们只去查询和拉取已经成熟发布的镜像资源。



首先我们打开 [docker hub](https://hub.docker.com/search?type=image) (由于国内网络问题，可能需要翻墙)。

dockerhub.png

接着我就搜索我想要的镜像也就是jdk8

dockerhub2.png

然后我们点击后，就可以查看到该镜像的具体细节。

dockerhub3.png

如果我们一开始不知道如何使用的话，我们可以将网页向下拉，找到具体介绍如何使用该镜像的文档。

dockerhub4.png

这样 按照文档上的来，我们就可以以 所查询的镜像作为base image ,在此环境上继续创建镜像。

当然我们也可以使用Dockerfile命令自己创建属于自己的base image 。这个就是在熟练使用之后的操作了。



#### LABEL

给创建的镜像添加标签，比如作者信息，版本信息，描述信息等。

```shell
LABEL maintainer = "作者姓名"

LABEL version = "1.0"

LABEL description = "描述"
```

 其实LABEL 比较像我们写代码时候的注释，在创建镜像是应该是必不可少的。



#### RUN

相当于执行命令，比如我们需要在镜像中安装一些软件，那么就需要使用RUN语法，但是需要注意的是，由于每条RUN操作会在docker之中新建一个镜像，所以我们尽量将一些命令合并成一条，在这里我们可以使用反斜杠换行，使其阅读美观。

```shell
RUN yum update && \
    yum install -y vim 
```



#### WORKDIR

类似于linux下的 `cd` 命令 

```shell
WORKDIR /test   #如果没有会自动创建test目录
WORKDIR demo    #同上
RUN pwd       # 输出 /test/demo
```

在对目录进行操作时，我们需要注意

- 尽量使用WORKDIR 不要使用 RUN cd 
- 尽量使用绝对路径



#### ADD 和 COPY

|          |                ADD                |           COPY           |
| :------: | :-------------------------------: | :----------------------: |
| 相同功能 |     将某文件复制到固定目录下      | 将某文件复制到固定目录下 |
| 不同功能 | 可以将tar文件解压提取到固定目录下 |       单纯复制文件       |
|   举例   | ADD test /    ADD test.tar.gz / | COPY test / |

注意： 如果想添加远程文件、目录，请使用 RUN + curl/weget



#### ENV

为当前容器设置环境变量时，我们可以使用ENV设置一个常量，比如我们想安装一个5.7的MYSQL 我们可以这么做

```shell
ENV MYSQL_VERSION 5.7   # 设置常量 MYSQL_VERSION  值是 5.7
RUN apt-get install -y mysql-server = “${MYSQL_VERSION}”  #引用常量
```



#### CMD和ENTRYPOINT

在介绍之前，我们要知道两种书写`Dockfile`命令行的格式，前提是大家要熟悉linux下的基本命令

##### shell格式

```shell
RUN apt-get install -y vim
CMD echo "hello docker"
ENTRYPOINT echo "hello docker"
```



##### Exec格式

```shell
RUN ["apt-get","install","-y","vim"]
CMD ["/bin/echo","hello docker"]
ENTRYPOINT ["/bin/echo","hello docker"]
```



我们来看一段命令

Dockerfile1:

```shell
FROM centos
ENV name hello-Docker
ENTRYPOINT echo "${name}" # 输出是 hello-world 
```



Dockerfile2

```shell
FROM centos
ENV name hello-Docker
ENTRYPOINT ["bin/echo","${name}"]  # 输出是 ${name}
```



出现两种不同输出的原因是因为 我们第一个Dockerfile 是shell格式执行命令时，默认就是 在linux的shell里执行。

但是第二个Dockerfile 我们使用的是Exec格式，我们去执行的是 echo这个命令而不是在linux的shell下执行命令，因此输出不一样。 



##### CMD

- 容器启动时默认执行的命令

- 如果docker run 指定了其他命令，CMD命令被忽略

  - 比如 执行这段Dockerfile

    ```shell
    FROM centos
    CMD echo "hello docker"
    ```

    docker run [image]  输出就是 hello docker

    docker run [image] /bin/bash  默认进入了当前container

- 如果定义了多个CMD,只有最后一个CMD会被执行。



##### ENTRYPOINT

- 让容易以应用程序或者服务的形式运行
- 不会被忽略，一定会执行
- 最佳实践：写一个shell 脚本作为 ENTRYPOINT 



#### EXPOSE

功能为暴漏容器运行时的监听端口给外部

但是EXPOSE并不会使容器访问主机的端口

如果想使得容器与主机的端口有映射关系，必须在容器启动的时候加上 -P参数



#### VOLUME

可实现挂载功能，容器告诉Docker在主机上创建一个目录(默认情况下是在/var/lib/docker),然后将其挂载到指定的路径。当删除使用该Volume的容器时,Volume本身不会受到影响,它可以一直存在下去。

比如 

```shell
FROM centos
VOLUME /test   #将该镜像的存储内容挂载到test文件夹下。这样即使删除了该镜像，再重新创建后，也不会影响数据
CMD echo "hello docker"
```



## 总结

只是列出了基本常用的命令，其他特殊的命令可以查看相关文档  [docker Dos](https://docs.docker.com/engine/reference/builder/)

dockerDocs.png