# Spring Cloud系列--Spring Cloud Config分布式配置中心

## 简介

> Spring Cloud Config provides server and client-side support for externalized configuration in a distributed system. With the Config Server you have a central place to manage external properties for applications across all environments. 
>
> Spring Cloud Config为分布式系统中的外部化配置提供服务器和客户端支持。使用Config Server，您可以在所有环境中管理应用程序的外部属性。



Spring Cloud Config 默认采用Git来存储配置信息，也同样支持其他存储方式比如 SVN,本地化文件系统等。接下来让我们使用Git存储来做分布式的配置文件管理。



## 构建配置中心

### 一.服务架构流程

flow.png



我们可以从图中看到，在向注册中心注册时我们的客户端需要从config-server中获取外部化配置，然后客户就可以随意向客户端发送请求了。

### 二.搭建项目

在这里我们使用IDEA默认提供的创建方式来创建配置中心。打开IDEA， File-> New Project->Spring Initializr 

点击next。

p1.png

p2.png



#### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.4.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>my-config-server</groupId>
    <artifactId>config-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>config-server</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR1</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>
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

</project>
```



### 三.Git仓库配置

相信各位程序员大大们对github都再熟悉不过了，我就不说如何配置和创建了。这里我在我的github中创建了一个名为`consumer-test.properties`文件。内容很简单。

```properties
hello=hello world come from neal github
```



##### 我们需要做的就是，使用客户端从Spring Cloud Config 服务中获取到该外部化配置。



### 四.整合配置

#### 1.配置文件`application.properties`

我们现在来将config-server和 Git仓库匹配上吧。

在config-server中，重要的配置都在`application.properties`

```properties
# 配置 Spring Cloud Config Server应用名
spring.application.name=config-server

#端口
server.port = 7070
#注册中心地址
eureka.client.service-url.defaultZone=http://localhost:1111/eureka/


#配置config
#git 仓库位置
spring.cloud.config.server.git.uri=https://github.com/NealLemon/myconfig
#git 搜索的相对路径
spring.cloud.config.server.git.search-paths=test

#git 仓库用户名以及密码
spring.cloud.config.server.git.username=你的github账号
spring.cloud.config.server.git.password=你的github密码
```



我们可以看到前三个配置再熟悉不过了，这里就不多解释了。我们来看一下之后的配置。

- `spring.cloud.config.server.git.uri`：配置Git仓库的位置，直接从浏览器地址栏中复制即可。

git1.png

- `spring.cloud.config.server.git.search-paths`：搜索配置文件的相对路径。这里就是上图中的**test**文件夹
- `spring.cloud.config.server.git.username/password`:Git仓库的帐号密码，这里可配可不配。



#### 2.引导类`ConfigServerApplication`

```java
//注册发现客户端
@EnableEurekaClient
//开启spring cloud config 自动装配
@EnableConfigServer
@SpringBootApplication
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }

}
```



### 五.启动

首先在启动Config-Server之前，我们需要一个注册中心，也就是我们之前文章所用的**spring-cloud-test-register**

如果不知道这个怎么搭建 ，可以参照之前的文章。

首先启动 注册中心（`spring-cloud-test-register`） -> 配置中心(`config-server`)

我们来看一下 注册中心，查看我们的配置中心是否注册到了 注册中心中。

p3.png



我们可以看到，已经注册进来了，我们接着来看一下我们外部化配置是否读取到了。在这里我们有一个规则，这个规则是我看  翟永超大神的《Spring Cloud 微服务实战》中摘录的

> /{application}/{profile}/[{label}]
>
>  /{application}-{profile}.yml
>
>  /{application}-{profile}.properties 
>
>  /{label}/{application}-{profile}.properties
>
> /{label}/{application}-{profile}.yml



这里的`label`  指的是对应Git上的不同的分支。默认为master。其余的可如图

p4.png



在这里我们也可以使用POSTMAN来查看配置的内容。

p5.png



到这里我们的配置中心已经搞定了。接下来让我们来搞一下客户端。



## 客户端配置（ribbon-consumer）

这里的客户端，我就使用之前创建的客户端了。很简单，如何搭建，可以回去看一下[Spring Cloud系列--简单实现Ribbon负载均衡](https://www.jianshu.com/p/e828b63a5cd8)。

#### 1.配置文件

`bootstrap.properties`

```properties
## 用户 Ribbon 客户端应用
spring.application.name=ribbon-consumer
server.port=9000

## 配置客户端应用关联的应用
spring.cloud.config.name = consumer
## 关联 profile
spring.cloud.config.profile = test
## 关联 label
spring.cloud.config.label = master
## 激活 Config Server 服务发现
spring.cloud.config.discovery.enabled = true
## Config Server 服务器应用名称
spring.cloud.config.discovery.serviceId = config-server
## Spring Cloud Eureka 客户端 注册到 Eureka 服务器
eureka.client.service-url.defaultZone=http://localhost:1111/eureka/
```



这里我们把之前`application.properties`中的内容全部删除。之所以配置`bootstrap.properties`这个是基于springboot的外部化配置加载顺序。具体的加载顺序可以查看相关文章，我也有过大概提过，是在这篇文章里

[Springboot--外部化配置(一)](https://www.jianshu.com/p/af104768ab3b)。我简单的debugger了一下，大体的加载顺序如下图。

p6.png



在配置中我们需要关注这么几个配置就可以了。其他的看注释就可以理解了。

- `spring.cloud.config.discovery.enabled`:激活 Config Server服务发现，这里默认是关闭的，如果不开启的话，无法从注册中心获取到相关配置信息。
- `spring.cloud.config.discovery.serviceId`：对应配置中心的服务ID 这里就是 config-server。我们看翟永超大神的书的时候，配置的时候并没有这个，而是直接把uri 配置上了。两种方式，大家选熟悉的来。



配置文件部分就完事了。



#### 2.添加依赖

pom.xml 中添加如下依赖

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
```



#### 3.添加测试方法

这里新建了一个controller，就不多做解释了 大家都能看懂

```java
/**
 以这种方式注释的Bean可以在运行时刷新，
 并且使用它们的任何组件将在下一个方法调用上获得一个新实例，
 完全初始化所有依赖项。
 */
@RefreshScope
@RestController
public class ConsumerConfigController {

    //获取外部化配置 key 为hello的键值
    @Value("${hello}")
    private String hello;

    
    @GetMapping("/hello")
    public String configHelloWorld1() {
        return this.hello;
    }
}
```



#### 4.启动并测试

由于之前启动了 注册中心和配置中心，所以我们现在只需要直接启动客户端就行。

我们启动程序会看到控制台打印

p7.png

可以看到，我们已经读取了 配置中心的配置。

我们打开注册中心页面，确认一下是否已经注册进来了。

p8.png

可以确认没问题。接下来就是见证奇迹的时刻。让我们测试一下是否可以读取到 配置中心的配置吧。



p9.png



结果很完美，读取并且输出了内容。



## 总结

分布式配置中心方便了我们整体项目的运维以及管理，使用好了可以使我们的项目开发部署测试都完美的进行。让我们继续加油了解吧，把学到的东西利用最大化。

