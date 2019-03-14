# Spring Cloud系列（二）-- 服务治理

## 为什么要使用服务治理?

​    设想一下，我们正在写代码使用了提供REST API或者Thrift API的服务，为了完成一次服务请求，代码需要知道服务实例的网络位置（IP地址和端口）。传统应用都运行在物理硬件上，服务实例的网络位置都是相对固定的。并且随着系统功能越来越复杂，相应的硬件配置也会越来越多比如集群配置，服务配置等。这些配置都需要手工去维护，这种维护工作不仅复杂而且工作量非常大。	

   因此在微服务中，服务治理就应运而生了。服务治理框架有很多，但是核心的功能就是两点，**服务注册**和**服务发现**。

- 服务注册：在服务治理框架中，通常都会构建一个注册中心，每个服务单元向注册中心登记自己提供的服务，包括服务的主机与端口号、服务版本号、通讯协议等一些附加信息。注册中心按照服务名分类组织服务清单，同时还需要以心跳检测的方式去监测清单中的服务是否可用，若不可用需要从服务清单中剔除，以达到排除故障服务的效果。
- 服务发现：在服务治理框架下，服务间的调用不再通过指定具体的实例地址来实现，而是通过服务名发起请求调用实现。服务调用方通过服务名从服务注册中心的服务清单中获取服务实例的列表清单，通过指定的负载均衡策略取出一个服务实例位置来进行服务调用。



## Netflix Eureka搭建高可用注册中心

  我们这里为了方便整合版本，我们直接使用IDEA来创建服务端。

### 创建Eureka Server

1.点击File 选择NEW -> Project

step1.png

2.点击下一步进入到创建页面

step2.png

3.点击下一步，选择Cloud Dicovery   -- Eureka Server

step3.png

4.点击下一步直至创建完成。



5.查看pom.xml以及maven依赖

pom.xml

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
    <groupId>com.cloud</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR1</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
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

    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
        </repository>
    </repositories>

</project>
```

step4.png



6.在启动引导类中加入注解	`@EnableEurekaServer`

```java
@EnableEurekaServer
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```



7.在resources文件夹下创建两个properties。

step5.png

application-peer1.properties

```properties
#服务ID
spring.application.name=eureka-server
#端口
server.port=1111
#实例主机名
eureka.instance.hostname=peer1
#是否注册自己
eureka.client.register-with-eureka=true
#是否需要检索服务
eureka.client.fetch-registry=true
#是否开启自我保护
eureka.server.enable-self-preservation=false
#默认URL
eureka.client.service-url.defaultZone=http://peer2:1112/eureka/
#是否启动健康诊断
eureka.client.healthcheck.enabled=true
```

application-peer2.properties

```properties
spring.application.name=eureka-server
server.port=1112

eureka.instance.hostname=peer2
eureka.client.register-with-eureka=true
eureka.client.fetch-registry=true
eureka.server.enable-self-preservation=false
eureka.client.service-url.defaultZone=http://peer1:1111/eureka/
eureka.client.healthcheck.enabled=true
```



注意由于是高可用注册中心，他们既是服务提供方也是服务消费方，所以对应的客户端URL（`eureka.client.service-url.defaultZone`）是指向对方的。



### 启动Eureka Server

1.前期工作已经做完了。我们现在需要将高可用注册中心启动起来，首先我们需要改变一下本地访问路径。

step6.png 

2.打开对应的文件。修改其内容。

step7.png

3.IDEA的启动配置修改

step8.png

4.添加springboot启动配置

step9.png

5.进行简单的配置，包括server的名称，启动类的选择，还有加载配置。

step10.png

step11.png

我们可以看到 两个除了启动参数不同之外，其他的都是一样的。

6.接下来让我们启动一下看看效果。

step12.png

我们可以看到节点分片（DS Replicas）中已经有了 peer2,同样，如果我们访问http://localhost:1112/，他的分片就是peer1。多节点的配置已经完成。

我们从图中 **Instances currently registered with Eureka** 下看到了这两个服务的注册信息，所以我们说高可用注册中心的服务提供者也是服务的消费者。



## 总结

  我们发现其实搭建 Eureka Server 其实不难，但是我们要做到知其然并知其所以然。



