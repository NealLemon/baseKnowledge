# Spring Cloud系列（一）--初识

## 微服务架构

​“微服务架构（*Microservice Architecture*）”一词在过去几年里广泛的传播，它用于描述一种设计应用程序的特别方式，作为一套独立可部署的服务。目前，这种架构方式还没有准确的定义，但是在围绕业务能力的组织、自动部（automated deployment）、端智能（intelligence in the endpoints）、语言和数据的分散控制，却有着某种共同的特征。	

  简而言之，微服务架构风格，就像是把一个单独的应用程序开发为一套小服务，每个小服务运行在自己的进程中，并使用轻量级机制通信，通常是 HTTP API。这些服务围绕业务能力来构建，并通过完全自动化部署机制来独立部署。这些服务使用不同的编程语言书写，以及不同数据存储技术，并保持最低限度的集中式管理。

我们可以使用一个例子来说明一下，这里引用了网上的例子，原文是系列文章，写的非常好，强烈建议大家看一下。[原文地址](https://www.nginx.com/blog/introduction-to-microservices/)    [翻译博客地址](http://www.dockone.io/people/hokingyang)

开发一款类似于滴滴打车的调度软件。核心业务模块包括:

- BILLING：计费模块
- PASSENGER MANAGERMENT: 乘客管理模块
- NOTIFICATION:身份认证模块
- PAYMENTS:订单模块
- TRIP MANAGERMENT: 定位功能管理模块
- DRIVER MANAGERMENT: 司机管理模块

围绕着核心的是与外界打交道的适配器，适配器包括数据库访问组件、生产和处理消息的消息组件，以及提供API或者UI访问支持的web模块等。

### 单体架构模式

singleapp.png

### 微服务架构模式

microservice.png

一个微服务一般完成某个特定的功能，每一个微服务都是微型六角形应用，都有自己的业务逻辑和适配器。一些微服务还会发布API给其它微服务和应用客户端使用。



具体的其他内容可以点击链接查看  [微服务（Microservices）翻译](http://blog.didispace.com/microservices-translate/)



## Spring Cloud

  Spring Cloud是一个相对比较新的微服务框架，虽然Spring Cloud时间最短, 但是相比Dubbo等RPC框架, Spring Cloud提供的全套的分布式系统解决方案。spring Cloud 为开发者提供了在分布式系统（配置管理，服务发现，熔断，路由，微代理，控制总线，一次性token，全居琐，决策竞选，分布式session，集群状态）中快速构建的工具，使用Spring Cloud的开发者可以快速的启动服务或构建应用．它们将在任何分布式环境中工作，包括开发人员自己的笔记本电脑，裸物理机的数据中心，和像Cloud Foundry云管理平台。

原英文出处：

```
Spring Cloud provides tools for developers to quickly build some of the common patterns in distributed systems (e.g. configuration management, service discovery, circuit breakers, intelligent routing, micro-proxy, control bus, one-time tokens, global locks, leadership election, distributed sessions, cluster state). Coordination of distributed systems leads to boiler plate patterns, and using Spring Cloud developers can quickly stand up services and applications that implement those patterns. They will work well in any distributed environment,  including the developer’s own laptop, bare metal data centres, and managed platforms such as Cloud Foundry.
```



Spring Cloud 专注于为典型的使用实例并为这些实例提供可扩展机制。

- 分布式/版本化配置
- 服务注册和发现
- 路由
- 服务之间的通信
- 负载均衡
- 断路器
- 全球锁
- 决策竞选和集群状态
- 分布式消息



### 版本说明

针对不同的spring-boot版本，对应的spring-cloud版本也是不同的。

| Release Train | Boot Version |
| :-----------: | :----------: |
| Boot Version  |    2.1.x     |
|   Finchley    |    2.0.x     |
|    Edgware    |    1.5.x     |
|    Dalston    |    1.5.x     |

