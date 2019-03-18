# Spring Cloud系列（二）-- Spring Cloud Eureka

## Eureka

### 核心要素

- 服务注册中心:提供服务注册与发现功能。
- 服务提供者：提供服务的应用。
- 服务消费者：消费者从注册中心获取服务提供者的服务列表，从而在消费时，知道去何处调用其所需要的服务。



### 服务治理

借用《Spring Cloud微服务实战》书上的图。

服务治理.png

- 注册中心
  - 服务同步
  - 失效剔除
  - 自我保护
- 服务提供者
  - 服务注册
  - 服务续约
- 服务消费者
  - 获取服务
  - 服务调用
  - 服务下线



## 服务端源码分析

源码分析我们就必须先从启动类来一步一步深入。

y1.png

点击`@EnableEurekaServer`注解，开始源码之旅。

y2.png

我们可以看到 `@EnableEurekaServer` 是一个模式注解，因此我们进入`@import`引入的类来查看。

y3.png

我们可以看到，这个类中什么东西都没有，只是一个标记，标记负责添加标记的bean已经激活。我们可以通过注解看到，真正激活的是`EurekaServerAutoConfiguration`这个类。我们继续往下看。

y4.png

果然，这个才是真正的EurekaServer配置Bean。其实我们知道Spring自己也有自己的一种命名规范，像这个类 我们不需要猜都知道是 Spring 自动装配的配置类。我为什么会这么说，我们先搜索一下Spring Boot的自动装配注解`EnableAutoConfiguration` 然后查找他所使用的地方，就可以看到很多类似的命名了。

y5.png

y6.png

我们查看标记的第一个 spring.factories

y7.png 

可以看到 spring boot 自动装配的很多bean 都是以AutoConfiguration结尾的。由此我们就可以推断出了 刚刚我们查看到的`EurekaServerAutoConfiguration`也是自动装配的。所以我们接着查看第二个spring.factories，也就是2.1.1.RELEASE\spring-cloud-netflix-eureka-server-2.1.1.RELEASE-sources.jar!\META-INF\spring.factories

y8.png

Spring boot的自动装配这里就不多做解释，感兴趣的同学可以自己去网上搜一下相关内容。



回归正题，让我们来主要看看 `EurekaServerAutoConfiguration`这个装配bean都有哪些主要的内容。

因为源码很长，所以我们先分开解释。

#### 注解类

```java
@Configuration
//引用了 EurekaServerInitializerConfiguration 类，相当于我们在Spring XML中配置了一个类
@Import(EurekaServerInitializerConfiguration.class)
//在Bean factory中 当存在EurekaServerMarkerConfiguration.Marker 实例时 加载
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
//外部化配置类引用
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
      InstanceRegistryProperties.class })
//自己的外部化配置引用
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter {
```



#### 配置Bean

```java
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter {

   //相关的bean的自动装配省略

   //首先判断外部配置中是否有 eureka.dashboard.enabled 的配置 如果没有也可以加载
   //注册EurekaController 提供服务端相关的API
   @Bean
   @ConditionalOnProperty(prefix = "eureka.dashboard", name = "enabled", matchIfMissing = true)
   public EurekaController eurekaController() {
      return new EurekaController(this.applicationInfoManager);
   }

   //注册 eureka注册服务实例bean 	
    //处理所有操作到对等节点的复制，以使它们保持同步。
   @Bean
   public PeerAwareInstanceRegistry peerAwareInstanceRegistry(
         ServerCodecs serverCodecs) {
      this.eurekaClient.getApplications(); // force initialization
      return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig,
            serverCodecs, this.eurekaClient,
            this.instanceRegistryProperties.getExpectedNumberOfClientsSendingRenews(),
            this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
   }

   //注册管理节点声明周期的bean
   @Bean
   @ConditionalOnMissingBean
   public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry,
         ServerCodecs serverCodecs) {
      return new RefreshablePeerEurekaNodes(registry, this.eurekaServerConfig,
            this.eurekaClientConfig, serverCodecs, this.applicationInfoManager);
   }

    //注册 EurekaServer 上下文
   @Bean
   public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
         PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
      return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
            registry, peerEurekaNodes, this.applicationInfoManager);
   }

   //注册 eurekaServer 的启动类 这里需要注意，这只是实例化了启动类，并没有启动eurekaServer
    //真正的调用是在类头中的注解 @Import(EurekaServerInitializerConfiguration.class) 
    //中的类进行启动的
   @Bean
   public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
         EurekaServerContext serverContext) {
      return new EurekaServerBootstrap(this.applicationInfoManager,
            this.eurekaClientConfig, this.eurekaServerConfig, registry,
            serverContext);
   }

   /**
    * 注册 Jersey拦截器
    */
   @Bean
   public FilterRegistrationBean jerseyFilterRegistration(
         javax.ws.rs.core.Application eurekaJerseyApp) {
      FilterRegistrationBean bean = new FilterRegistrationBean();
      bean.setFilter(new ServletContainer(eurekaJerseyApp));
      bean.setOrder(Ordered.LOWEST_PRECEDENCE);
      bean.setUrlPatterns(
            Collections.singletonList(EurekaConstants.DEFAULT_PREFIX + "/*"));

      return bean;
   }

  //使用Eureka服务器所需的所有资源构建Jersey
   @Bean
   public javax.ws.rs.core.Application jerseyApplication(Environment environment,
         ResourceLoader resourceLoader) {

      ClassPathScanningCandidateComponentProvider provider = new ClassPathScanningCandidateComponentProvider(
            false, environment);

      // Filter to include only classes that have a particular annotation.
      //
      provider.addIncludeFilter(new AnnotationTypeFilter(Path.class));
      provider.addIncludeFilter(new AnnotationTypeFilter(Provider.class));

      // Find classes in Eureka packages (or subpackages)
      //
      Set<Class<?>> classes = new HashSet<>();
      for (String basePackage : EUREKA_PACKAGES) {
         Set<BeanDefinition> beans = provider.findCandidateComponents(basePackage);
         for (BeanDefinition bd : beans) {
            Class<?> cls = ClassUtils.resolveClassName(bd.getBeanClassName(),
                  resourceLoader.getClassLoader());
            classes.add(cls);
         }
      }

      // Construct the Jersey ResourceConfig
      Map<String, Object> propsAndFeatures = new HashMap<>();
      propsAndFeatures.put(
            // Skip static content used by the webapp
            ServletContainer.PROPERTY_WEB_PAGE_CONTENT_REGEX,
            EurekaConstants.DEFAULT_PREFIX + "/(fonts|images|css|js)/.*");

      DefaultResourceConfig rc = new DefaultResourceConfig(classes);
      rc.setPropertiesAndFeatures(propsAndFeatures);

      return rc;
   }

   //私有的 EurekaServer 配置类
   @Configuration
   protected static class EurekaServerConfigBeanConfiguration {

      //注册服务端的注册配置信息bean  
       //比如 eureka.server.XXX   配置的信息
      @Bean
      @ConditionalOnMissingBean
      public EurekaServerConfig eurekaServerConfig(EurekaClientConfig clientConfig) {
         EurekaServerConfigBean server = new EurekaServerConfigBean();
         if (clientConfig.shouldRegisterWithEureka()) {
            // Set a sensible default if we are supposed to replicate
            server.setRegistrySyncRetries(5);
         }
         return server;
      }

   }

   //调用刷新方法时 更新节点信息
    //实现应用程序事件监听接口，用来实现更新节点操作
   static class RefreshablePeerEurekaNodes extends PeerEurekaNodes
         implements ApplicationListener<EnvironmentChangeEvent> {

      RefreshablePeerEurekaNodes(final PeerAwareInstanceRegistry registry,
            final EurekaServerConfig serverConfig,
            final EurekaClientConfig clientConfig, final ServerCodecs serverCodecs,
            final ApplicationInfoManager applicationInfoManager) {
         super(registry, serverConfig, clientConfig, serverCodecs,
               applicationInfoManager);
      }

       //监听事件， 更新节点信息
      @Override
      public void onApplicationEvent(final EnvironmentChangeEvent event) {
         if (shouldUpdate(event.getKeys())) {
            updatePeerEurekaNodes(resolvePeerUrls());
         }
      }

      //更新条件判断 
      //仅当eureka.client.use-dns-for-fetching-service-urls 为false时，并且
      // eureka.client.availability-zones
       //eureka.client.region
       //eureka.client.service-url.&lt;zone&gt; 这几个配置改变时 ，更新
      protected boolean shouldUpdate(final Set<String> changedKeys) {
         assert changedKeys != null;

         // if eureka.client.use-dns-for-fetching-service-urls is true, then
         // service-url will not be fetched from environment.
         if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
            return false;
         }

         if (changedKeys.contains("eureka.client.region")) {
            return true;
         }

         for (final String key : changedKeys) {
            // property keys are not expected to be null.
            if (key.startsWith("eureka.client.service-url.")
                  || key.startsWith("eureka.client.availability-zones.")) {
               return true;
            }
         }
         return false;
      }

   }
}
```



## 总结

  由于是自己跟的源码，可能理解的也有问题，并且目前只是看了 EurekaServerAutoConfiguration 的源码，之后会讨论 导入类 `EurekaServerInitializerConfiguration` 。

