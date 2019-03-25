# Spring Cloud系列--简单实现Ribbon自定义负载均衡

  Ribbon是客户端的负载均衡，在微服务调用，API网关的请求转发等，都离不开Ribbon。因此理解和使用Spring-cloud-Ribbon 是非常重要的。我们先来看一下如何使用Ribbon进行负载均衡。

我们将使用4个服务来实现。

- 服务注册
- 服务提供1
- 服务提供2
- 客户端



具体的架构流程图如下

ribbon-server.png



## 项目搭建

具体搭建方法参照[Spring Cloud系列-- 服务治理简单搭建](https://www.jianshu.com/p/008168cb0c97) ，我们使用IDEA简单搭建一下4个服务。这里只给出主要的截图和代码。

### 注册中心

#### 新建项目

register1.png

web.png

register2.png

#### 启动配置

##### application.properties

```properties
#服务ID
spring.application.name=eureka-server
#端口
server.port=1111
#实例主机名
eureka.instance.hostname=localhost
#是否注册自己
eureka.client.register-with-eureka=false
#是否需要检索服务
eureka.client.fetch-registry=false
#是否开启自我保护
eureka.server.enable-self-preservation=false

eureka.client.service-url.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
```



##### pom.xml

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
    <groupId>com.register</groupId>
    <artifactId>spring-cloud-test-register</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>spring-cloud-test-register</name>
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
            <artifactId>spring-boot-starter-web</artifactId>
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

##### SpringCloudTestRegisterApplication

```java
@EnableEurekaServer
@SpringBootApplication
public class SpringCloudTestRegisterApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudTestRegisterApplication.class, args);
    }

}
```



##### 直接启动测试

register3.png



### 服务提供

#### 新建项目

provider1.png

web.png

provider2.png



#### 启动配置

这里我们参考之前的服务治理搭建，搞出两个外部化配置文件，通过IDEA的启动配置来启动两个服务，如果不知道怎么配置，可以参考之前那篇文章。

##### application-provider1.properties

```properties
spring.application.name=ribbon-service
#端口
server.port=1112
#指向注册中心
eureka.client.service-url.defaultZone=http://localhost:1111/eureka/
```



##### application-provider2.properties

```properties
spring.application.name=ribbon-service
#端口
server.port=1113
#指向注册中心
eureka.client.service-url.defaultZone=http://localhost:1111/eureka/
```



##### pom.xml

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
    <groupId>com.ribbonprovider</groupId>
    <artifactId>provider</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>provider</name>
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
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



##### com.ribbonprovider.provider.HelloWorldController

```java
@RestController
public class HelloWorldController {

    //获取当前端口配置
    @Value("${server.port}")
    private String port;

    @GetMapping("/hello")
    public String helloWorld() {
        return "Hello World from port:" + port;
    }
}
```



##### com.ribbonprovider.provider.ProviderApplication

```java
@EnableDiscoveryClient
@SpringBootApplication
public class ProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProviderApplication.class, args);
    }

}
```



#### 启动查看注册结果

provider3.png



我们可以看到两个服务已经注册到了注册中心上。



### 服务消费（客户端）

#### 新建项目

consumer1.png

consumer2.png

这里就新加入了 ribbon依赖。



#### 启动配置

##### application-consumer.properties

```properties
spring.application.name=ribbon-consumer
server.port=1114
eureka.client.service-url.defaultZone=http://localhost:1111/eureka/
```



##### pom.xml

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
    <groupId>com.consumer</groupId>
    <artifactId>myribbon</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>myribbon</name>
    <description>Demo project for Spring Boot</description>

    <properties>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.SR1</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
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



##### com.consumer.myribbon.HelloConsumerController

```java
@RestController
public class HelloConsumerController {

    @Autowired
    RestTemplate restTemplate;

    @GetMapping("/helloworld")
    public String helloWorld() {
        return restTemplate.getForEntity("http://RIBBON-SERVICE/hello",String.class).getBody();
    }
}
```



##### com.consumer.myribbon.MyribbonApplication

```java
@EnableEurekaClient
@SpringBootApplication
public class MyribbonApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyribbonApplication.class, args);
    }

    @Bean
    //负载均衡注解
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
```



#### 启动查看注册结果

consumer3.png



#### 请求服务查看负载均衡结果

我们直接请求本地路径 

http://localhost:1114/helloworld  



来查看返回信息中的值

consumer4.png

consumer5.png



可以看到端口的变化情况。



## 总结

 在Spring强大的生态下，一切都变得很简单，希望大家不要局限于表面，尽量明白Ribbon的原理。