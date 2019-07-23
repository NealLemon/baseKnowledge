# Spring Cloud系列-gateway整合

​	由于最近跳槽到了专门搞微服务的部门，了解了很多微服务相关的知识，由于接触的东西和学习公司相关组件耽误了很久，现在才更新，对于网关，很多公司都会基于netty来实现自己的网关。因此在我们自己练习或者实现网关的时候可以借鉴gateway，但是拿gateway直接做 路由，还是比较麻烦。关于Gateway的概念，网上文章已经解释的很清楚了。

## 重新整合服务模块

之前由于是自己练习，模块部署的很乱，然后这段时间自己重新写的时候，发现有些模块找不到了（手动捂脸），所以就重新整合了各个服务模块。之前有些config等模块，就不参与整合了。

架构.png



### 注册中心（demo-register）

##### 目录

注册中心目录.png

##### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>demo-register</groupId>
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

</project>
```

##### yml文件(application.yml)

```yml
server:
  port: 1111  #端口
spring:
  application:
    name: demo-register  #应用名
eureka:
  instance:
    hostname: localhost   #实例主机名
  client:
    fetch-registry: false  #是否需要检索服务
    register-with-eureka: false  #是否注册自己
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/   #注册中心地址
  server:
    enable-self-preservation: false  #是否开启自我保护
```

##### 启动类(DemoApplication)

```java
package demoregister.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer   //声明注册中心服务
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```

### 服务提供者(demo-provider)

##### 目录

服务提供者目录.png

##### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>demo-provider</groupId>
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
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

##### yml(application-p1.yml)

```yaml
server: 
  port: 8088   #p2可以改为8089
spring:
  application:
    name: demo-provider
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:1111/eureka/
```

##### 服务提供接口 --Controller

```java
package demoprovider.demo.controller;


import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloWorldController {

    /**
     * 获取当前服务端口
     */
    @Value("${server.port}")
    private String port;

    /**
     * 简单的测试 rest接口
     * @param name
     * @return
     */
    @GetMapping("/hello/{name}")
    public String hello(@PathVariable("name") String name) {
        return "Hello {" + name + " } from port : " + port;
    }
}
```

##### 启动类

```java
package demoprovider.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@EnableDiscoveryClient  //服务客户端注册
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```



### 服务消费者(demo-consumer)

##### 目录

服务消费者目录.png

##### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>demo-consumer</groupId>
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
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



##### yml

```yaml
server:
  port: 8082
spring:
  application:
    name: demo-consumer
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:1111/eureka/
```



##### Feign调用接口

```java
/**
 * Feign接口
 */
@FeignClient("demo-provider")  //定义指定的服务提供应用名
public interface HelloWorldService {
    /**
     * 对应接口方法调用
     * @param name
     * @return
     */
    @GetMapping("/hello/{name}")
    String hello(@PathVariable("name") String name);
}
```

##### 对外调用接口--controller

```java
@RestController
public class HelloWorldController {


    /**
     * 自动装配Feign接口
     */
    @Autowired
    private HelloWorldService helloWorldService;

    /**
     * 通过请求路径来通过Feign调用 服务端的方法
     * @param name
     * @return
     */
    @GetMapping("/consumer/{name}")
    public String helloWorld(@PathVariable("name") String name) {
        return helloWorldService.hello(name);
    }
}
```

##### 启动类

```java
@SpringBootApplication
@EnableFeignClients(clients = HelloWorldService.class)   //申明HelloWorldService作为Feign的调用
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```



### 路由Gateway (demo-gateway)

##### 目录

gateway目录.jpg

##### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>gateway-demo</groupId>
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
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



##### yml

```yaml
server:
  port: 9898
spring:
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true  #从注册中心发现
      routes:
       - id: mygateway
         uri: lb://DEMO-CONSUMER  #服务端 service_id
         predicates:
          - Path=/consumer/**    #路由映射路径 这里注意，这个路径会追加到 service_id后
  application:
    name: demo-gateway
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:1111/eureka/
```



##### 启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```



## 整合后测试

以上简单的demo 关键部分我已经注释了。我们做一下启动整合测试。

#### 启动(注意启动顺序)

1. demo-register
2. demo-provider1、demo-provider2
3. demo-consumer
4. demo-gateway

关于如何使用两个不同的yml文件启动，可以参照我之前的文章。



#### 使用POSTMAN测试

postman结果.jpg



## 小结

  由于最近工作的感慨，觉得在工作的需求中阅读源码的效率和记忆才是最高的，因此以后如果在工作中用到相关组件的源码修改，会总结分享出来。虽然是简简单单的搭建，但是对于微服务新手，在spring cloud中也会踩到很多坑，希望有所帮助。



