# 使用Docker部署第一个Springboot应用

前面介绍了docker到底是什么？ 还有如何在centos中安装docker。那么现在让我们自己来实战一下，如何在docker上部署自己第一个项目。

## 创建一个Springboot程序

##### 1.初始化项目

我们打开 spring官方提供的初始化springboot项目页面 [Spring Initializr](https://start.spring.io/)。Dependencies选择web就可以了，项目如图:

spring-initialize.png



##### 2.使用IDEA或者ecplipse导入，我这里使用的是IDEA。

将pom.xml文件修改如下:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
   <modelVersion>4.0.0</modelVersion>
   <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>2.1.3.RELEASE</version>
      <relativePath/> <!-- lookup parent from repository -->
   </parent>
   <groupId>com.docker.example</groupId>
   <artifactId>demo</artifactId>
   <version>docker</version>
   <name>demo</name>
   <description>Demo project for Spring Boot</description>
   <packaging>jar</packaging>

   <properties>
      <java.version>1.8</java.version>
   </properties>

   <dependencies>
      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-web</artifactId>
      </dependency>

      <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-test</artifactId>
         <scope>test</scope>
      </dependency>
   </dependencies>

   <build>
      <plugins>
         <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
         </plugin>
      </plugins>
   </build>

</project>
```



##### 3.修改application.properties 文件

自定义请求路径

```properties
server.servlet.context-path=/test
```



##### 4.创建测试controller

```java
package com.docker.example.demo.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @ClassName DockerTestController
 * @Description docker-demo简单的controller
 * @Author Neal
 * @Date 2019/2/21 18:38
 * @Version 1.0
 */
@RestController
public class DockerTestController {

    @GetMapping("/docker")
    public String dockerTest() {
        return "hello docker";
    }
}
```



##### 5.最终项目目录

demo.png



##### 6.使用IDEA启动，测试项目是否部署成功。

ideastart.png

请求本地路径，查看返回结果

ideatest.png

我们看到项目没什么问题，那么现在让我们把springboot程序打包，打成jar包即可。



##### 7.打包

这里我使用IDEA中的MAVEN插件打包，非常简单快捷，但是就是需要走一遍测试。步骤如下：

maven.png

双击package即可，最后只要等待控制台输出SUCCESS即可。

success.png



我们会在项目中的`dockertest\demo\target\demo-docker.jar` 路径中看到自己打包的jar。

demojar.png



## 使用Docker部署（linux）

我这里使用的是自己的阿里云服务器，如果有本地虚拟机使用的centos系统也可以，这个大家可以自行选择。

##### 1.把jar包放到固定目录下

我的目录是 `/home/docker/docker-demo`

##### 2.创建`Dockerfile` 文件

使用`vim Dockerfile`  命令创建文件并将以下内容copy进你自己的Dockerfile文件中。

```shell
#获取base image
FROM adoptopenjdk/openjdk8:latest 
#类似于执行 linux指令
RUN mkdir /opt/app  
#类似于linux copy指令
COPY demo-docker.jar /opt/app/       
#执行命令 java -jar /opt/app/demo-docker.jar
CMD ["java", "-jar", "/opt/app/demo-docker.jar"] 
```

可能有同学会问 这命令是干什么的，稍后会有文章单独介绍。我们今天主要是实现docker部署一个springboot项目。

copy复制完后，按ESC并输入 `:wq`保存文件。

##### 3.在当前路径输入命令 `ls` 如果出现以下输出，表示正确。

ls.png

##### 4.创建docker 镜像。

```shell
sudo docker build -t docker-demo .
```

这里稍微解释一下 `build` 是创建命令 ，`-t` 是指定target 名称，    `docker-demo` 就是镜像名称  ,`.` 指的是在当前目录下 寻找 `Dockerfile`文件。 

执行以上指令，如果打印输出如下，表示创建成功。

image_success.png

##### 5.查看当前镜像列表

image-list.png



##### 6.生成container

执行以下命令

```shell
docker run -it -p 8080:8080 docker-demo
```

这里也稍微做一下解释 `run` 运行镜像 `-it`以交互模式运行容器并为容器重新分配一个伪输入终端  `-p` 端口映射，格式为：主机(宿主)端口:容器端口 。  最后的就是我们刚刚创建的镜像名称。

如果输出以下内容，表示部署基本成功。

container_output.png

##### 7.检测部署是否成功

这里我使用的是我自己阿里云服务器上的公网IP，大家可以选择自己的对外IP进行测试。我这里使用的是chrome浏览器。

docker测试.png



## 总结

一个简单的Springboot项目，已经使用docker部署完了。在部署这个小项目的时候，自己做过很多测试，包括基础镜像的创建，Dockerfile的调试等，如果各位对docker感兴趣，可以使用 [play with docker](https://labs.play-with-docker.com/) 来熟悉或者练习，具体怎么使用可以自行百度。





