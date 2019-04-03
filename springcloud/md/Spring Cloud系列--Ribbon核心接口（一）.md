# Spring Cloud系列--Ribbon源码（一）

  简单的实现了Spring Cloud Ribbon负载均衡，让我们大体了解一下Ribbon的几个核心接口。

## 核心接口总览

#### 实际请求客户端

- LoadBalancerClient
  - RibbonLoadBalancerClient



#### 负载均衡上下文

- LoadBalancerContext
  - RibbonLoadBalancerContext



#### 负载均衡器

- ILoadBalancer
  - NoOpLoadBalancer
  - BaseLoadBalancer
  - DynamicServerListLoadBalancer
  - ZoneAwareLoadBalancer



#### 负载均衡规则

- IRule
  - AbstractLoadBalancerRule
    - ClientConfigEnabledRoundRobinRule   --客户端配置
      - BestAvailableRule    --最可用规则
      - PredicateBasedRule   --一定逻辑的规则
        - ZoneAvoidanceRule  --区域过滤
        - AvailabilityFilteringRule   --可用性过滤
    - RoundRobinRule  --轮询规则
      - WeightedResponseTimeRule  --权重规则
    - RandomRule  --随即规则
    - RetryRule  --重试实现



## 从@LoadBalanced开始我们的Ribbon负载均衡

在上面一篇实现负载均衡的文章中，我们有这么一段代码。

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```



我们学习ribbon都知道，在使用RestTemplate时，要想实现负载均衡，需要添加@LoadBalanced注解。那么为什么需要添加这个注解呢？

我们先来看一下这个注解的源码

```java
/**
 * Annotation to mark a RestTemplate bean to be configured to use a LoadBalancerClient.
 * @author Spencer Gibb
 */
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {

}
```

通过注解我们可以看到这个注解被@Qualifier标识，学过spring的相信大家都知道这个注解的意思。我们通过这个注解的源码解释可以大体了解到

@Qualifier:这个注解可以用在参数或者域和属性上，在Spring自动装配时用于对候选beans的预选。

因此 我们只需要把@LoadBalanced 看做一个特殊的@Qualifier就可以了。



## Spring-Cloud-Ribbon的自动装配

了解了@LoadBalanced的作用，那么我就来看看@LoadBalanced在Ribbon配置中是如何使用的。



### RibbonAutoConfiguration的自动装配

按照正常套路，我们来查看对应包下的spring.factories

`spring-cloud-netflix-ribbon-2.1.1.RELEASE.jar!\META-INF\spring.factories`

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration
```

#### 注解部分

我们可以看到了自动装配了RibbonAutoConfiguration类。

同样我们先从注解部分看起



```java
@Configuration  
//前置条件注解，当满足时才会注册RibbonAutoConfiguration
@Conditional(RibbonAutoConfiguration.RibbonClassesConditions.class)   
//整合注册配置使用@RibbonClient注解的类
@RibbonClients
//在EurekaClientAutoConfiguration自动装配后才装配
@AutoConfigureAfter(name = "org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration")
//在下面两个类之前进行装配
@AutoConfigureBefore({ LoadBalancerAutoConfiguration.class,
      AsyncLoadBalancerAutoConfiguration.class })
//所支持的外部化配置
@EnableConfigurationProperties({ RibbonEagerLoadProperties.class,
      ServerIntrospectorProperties.class })
public class RibbonAutoConfiguration {
```

这里我发现了一个很有趣的地方，之前没有注意到就是

```java
@Conditional(RibbonAutoConfiguration.RibbonClassesConditions.class)  
```

我们都知道这是一个条件注解，但是我们看一下这个条件类的使用。

```java
/**
 * {@link AllNestedConditions} that checks that either multiple classes are present.
 */
static class RibbonClassesConditions extends AllNestedConditions {

   RibbonClassesConditions() {
      super(ConfigurationPhase.PARSE_CONFIGURATION);
   }

   @ConditionalOnClass(IClient.class)
   static class IClientPresent {

   }

   @ConditionalOnClass(RestTemplate.class)
   static class RestTemplatePresent {

   }

   @ConditionalOnClass(AsyncRestTemplate.class)
   static class AsyncRestTemplatePresent {

   }

   @ConditionalOnClass(Ribbon.class)
   static class RibbonPresent {

   }

}
```



这里用到了AllNestedConditions条件判断的父类。这个父类的作用就是创造符合的匹配条件，在类中标识所有`@Conditional`相关注解的嵌套条件类匹配时匹配。那么在这里我们看到的`RibbonClassesConditions`符合匹配类，就是在其内部所有条件匹配类符合条件时才会通过。进而 `RibbonAutoConfiguration`才会自动装配。



关于注解中的

```java
//在下面两个类之前进行装配
@AutoConfigureBefore({ LoadBalancerAutoConfiguration.class,
      AsyncLoadBalancerAutoConfiguration.class })
```

之后会有介绍。



关于这个注解，在Eureka Client已经给出解释。这里不再做叙述。

```java
@AutoConfigureAfter(name = "org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration")
```



##### 代码部分

```java
public class RibbonAutoConfiguration {

   //省略部分代码

    /**
    * 创建client load balancer client configuration实例的工厂
    */
   @Bean
   public SpringClientFactory springClientFactory() {
      SpringClientFactory factory = new SpringClientFactory();
      factory.setConfigurations(this.configurations);
      return factory;
   }

   
   /**
   * 创建RibbonLoadBalancerClient 用于客户端的负载均衡
   */
   @Bean
   @ConditionalOnMissingBean(LoadBalancerClient.class)
   public LoadBalancerClient loadBalancerClient() {
      return new RibbonLoadBalancerClient(springClientFactory());
   }

   /**
   * 重试策略工厂类
   */
   @Bean
   @ConditionalOnClass(name = "org.springframework.retry.support.RetryTemplate")
   @ConditionalOnMissingBean
   public LoadBalancedRetryFactory loadBalancedRetryPolicyFactory(
         final SpringClientFactory clientFactory) {
      return new RibbonLoadBalancedRetryFactory(clientFactory);
   }

   /**
   * 核心组件工厂
   */
   @Bean
   @ConditionalOnMissingBean
   public PropertiesFactory propertiesFactory() {
      return new PropertiesFactory();
   }

   /**
   *  负责初始化与spring cloud关联的子上下文
   */
   @Bean
   @ConditionalOnProperty("ribbon.eager-load.enabled")
   public RibbonApplicationContextInitializer ribbonApplicationContextInitializer() {
      return new RibbonApplicationContextInitializer(springClientFactory(),
            ribbonEagerLoadProperties.getClients());
   }

}
```



### LoadBalancerAutoConfiguration的自动装配

首先我们先来确定一下LoadBalancerAutoConfiguration在spring-cloud-commons-2.1.1.RELEASE.jar!\META-INF\spring.factories中配置的。

```properties
# AutoConfiguration
#省略部分
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.client.loadbalancer.AsyncLoadBalancerAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.LoadBalancerAutoConfiguration,\

```

我们主要来看LoadBalancerAutoConfiguration就可以了。其实两个自动装配是异曲同工的。



#### 注解部分

```java
@Configuration
//路径下有RestTemplate类
@ConditionalOnClass(RestTemplate.class)
//是否有LoadBalancerClient实例注册在上下文中
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {
```

注解没什么可说的 很简单。



#### 代码部分

```java
public class LoadBalancerAutoConfiguration {

   //step 1
   @LoadBalanced
   @Autowired(required = false)
   private List<RestTemplate> restTemplates = Collections.emptyList();

   @Autowired(required = false)
   private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

    //step4
   @Bean
   public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
         final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
      return () -> restTemplateCustomizers.ifAvailable(customizers -> {
         for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
            for (RestTemplateCustomizer customizer : customizers) {
               customizer.customize(restTemplate);
            }
         }
      });
   }

   //step2 
   @Bean
   @ConditionalOnMissingBean
   public LoadBalancerRequestFactory loadBalancerRequestFactory(
         LoadBalancerClient loadBalancerClient) {
      return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
   }

   @Configuration
   @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
   static class LoadBalancerInterceptorConfig {

       //step3
      @Bean
      public LoadBalancerInterceptor ribbonInterceptor(
            LoadBalancerClient loadBalancerClient,
            LoadBalancerRequestFactory requestFactory) {
         return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
      }

       //step5
      @Bean
      @ConditionalOnMissingBean
      public RestTemplateCustomizer restTemplateCustomizer(
            final LoadBalancerInterceptor loadBalancerInterceptor) {
         return restTemplate -> {
            List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                  restTemplate.getInterceptors());
            list.add(loadBalancerInterceptor);
            restTemplate.setInterceptors(list);
         };
      }
   }

}
```



之前的`RibbonAutoConfiguration`的自动装配就是为这个`LoadBalancerAutoConfiguration`准备组件的。因此我们需要好好的看看这个自动装配类。



##### Step1

```java
   //step 1
   @LoadBalanced
   @Autowired(required = false)
   private List<RestTemplate> restTemplates = Collections.emptyList();
```

在开始讲到的`@LoadBalanced`注解的作用，这里就体现出来了，我们使用了自动装配将标注`@LoadBalanced`的

RestTemplate实例都自动装到了List集合中。至于为什么装到集合中，我们稍后就知道了。



##### step2 

```java
   @Bean
   @ConditionalOnMissingBean
   public LoadBalancerRequestFactory loadBalancerRequestFactory(
         LoadBalancerClient loadBalancerClient) {
      return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
   }
```

负载均衡请求工厂类，为`LoadBalancerInterceptor`和`RetryLoadBalancerInterceptor`封装request请求。



##### Step3

```java
      @Bean
      public LoadBalancerInterceptor ribbonInterceptor(
            LoadBalancerClient loadBalancerClient,
            LoadBalancerRequestFactory requestFactory) {
         return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
      }
```

负载均衡拦截器组件。负责拦截使用`RestTemplate`发送的Http请求。



##### Step4 & Step5

这两段代码需要连起来看，是因为在Step4代码中的使用lambda表达式来调用Step5中的方法来 将每一个注解

`@LoadBalanced`的RestTemplate添加拦截器。

Step4:

```java
@Bean
public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
      final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
   return () -> restTemplateCustomizers.ifAvailable(customizers -> {
      for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
         for (RestTemplateCustomizer customizer : customizers) {
            customizer.customize(restTemplate);  //lambda调用Step5中的RestTemplateCustomizer Bean方法
         }
      }
   });
}
```

Step5:

```java
    @Bean
   @ConditionalOnMissingBean
   public RestTemplateCustomizer restTemplateCustomizer(
         final LoadBalancerInterceptor loadBalancerInterceptor) {
      return restTemplate -> {
         List<ClientHttpRequestInterceptor> list = new ArrayList<>(
               restTemplate.getInterceptors());
         list.add(loadBalancerInterceptor);
         restTemplate.setInterceptors(list);
      };
   }
}
```





## 总结

  Ribbon的基本的组件装配已经搞完了。至于Step4，Step5的Lambda表达式应用的很难理解。还需要自己去深琢磨。