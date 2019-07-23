# Spring Cloud系列--Spring Cloud Zuul网关服务

## 简介

> Zuul is the front door for all requests from devices and web sites to the backend of the Netflix streaming application. As an edge service application, Zuul is built to enable dynamic routing, monitoring, resiliency and security.                                                ----- 摘自[github](https://github.com/Netflix/zuul/wiki)

> Zuul是设备以及网站的所有请求从前端访问Netflix应用后端的前门。 作为边缘服务应用程序，Zuul旨在实现动态路由，监控，弹性和安全性。



简而言之就是一种API网关服务。

> API网关是一个更为智能的应用服务器，它的定义类似于面向对象设计模式中的Facade模式，它的存在就像是整个微服务架构系统的门面一样，所有的外部客户端访问都需要经过它来进行调度和过滤。它除了要实现请求路由，负载均衡，校验过滤功能之外，还需要更多能力，比如与服务治理框架结合、请求转发时的熔断机制、服务的聚合等一系列高级功能。                           ---摘自 《Spring Cloud微服务实战》



## Zuul服务网关

### 一、服务架构

structure.png



我们接下来要实现的就是当发送请求时，我们首先要在API网关做路由然后再由API网关转发到对应的服务上。同时在微服务应用注册时，从配置中心读取相关配置。

了解了大体流程，其实实现起来非常简单，毕竟只是应用不是深入研究。



### 二、Zuul服务搭建

在这里我们使用IDEA默认提供的创建方式来创建配置中心。打开IDEA， File-> New Project->Spring Initializr 

点击next。

p1.png



#### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.5.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>myzuul</groupId>
    <artifactId>my-zuul</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>my-zuul</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR1</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>
                
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

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

