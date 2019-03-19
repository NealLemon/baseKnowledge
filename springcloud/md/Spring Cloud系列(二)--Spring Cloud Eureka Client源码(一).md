# Spring Cloud系列(二)--Spring Cloud Eureka Client源码(一)

  在之前简单介绍了Eureka Server的源码,我们接下来可以看看 Eureka Client端的源码。跟之前一样先从启动类的注解开始。

## 注解  @EnableDiscoveryClient

 首先我们进入到注解中。查看一下源码。

```java
/**
 * Annotation to enable a DiscoveryClient implementation.
 * @author Spencer Gibb
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
//基于驱动编程方式的引入
@Import(EnableDiscoveryClientImportSelector.class)  
public @interface EnableDiscoveryClient {

   //开启自动注册
   /**
    * If true, the ServiceRegistry will automatically register the local server.
    * @return - {@code true} if you want to automatically register.
    */
   boolean autoRegister() default true;

}
```

可以看到，这个注解就是简单的开启服务自动注册的功能。



我们再来看一下引入的这个类`EnableDiscoveryClientImportSelector.class`的源码。

```java
@Order(Ordered.LOWEST_PRECEDENCE - 100)
public class EnableDiscoveryClientImportSelector
      extends SpringFactoryImportSelector<EnableDiscoveryClient> {

   @Override
   public String[] selectImports(AnnotationMetadata metadata) {
      String[] imports = super.selectImports(metadata);

      AnnotationAttributes attributes = AnnotationAttributes.fromMap(
            metadata.getAnnotationAttributes(getAnnotationClass().getName(), true));
      //从EnableDiscoveryClient注解中获取是否开启自动注册
      boolean autoRegister = attributes.getBoolean("autoRegister");

       //如果是，则将AutoServiceRegistrationConfiguration 进行加载
      if (autoRegister) {
         List<String> importsList = new ArrayList<>(Arrays.asList(imports));
         importsList.add(
               "org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration");
         imports = importsList.toArray(new String[0]);
      }
      else {
          //如果不是，则从外部化配置中，获取相关的配置参数
         Environment env = getEnvironment();
         if (ConfigurableEnvironment.class.isInstance(env)) {
            ConfigurableEnvironment configEnv = (ConfigurableEnvironment) env;
            LinkedHashMap<String, Object> map = new LinkedHashMap<>();
            map.put("spring.cloud.service-registry.auto-registration.enabled", false);
            MapPropertySource propertySource = new MapPropertySource(
                  "springCloudDiscoveryClient", map);
            configEnv.getPropertySources().addLast(propertySource);
         }

      }

      return imports;
   }

   @Override
   protected boolean isEnabled() {
      return getEnvironment().getProperty("spring.cloud.discovery.enabled",
            Boolean.class, Boolean.TRUE);
   }

   @Override
   protected boolean hasDefaultFactory() {
      return true;
   }

}
```

我们看到这个引用类中也就是准备装配`AutoServiceRegistrationConfiguration`配置类。我们可以在进一步看看这个类的源码，看看究竟到底是要装配什么。

`org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationConfiguration`

```java
@Configuration
@EnableConfigurationProperties(AutoServiceRegistrationProperties.class)
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
public class AutoServiceRegistrationConfiguration {
}
```

看到上面的源码 我也很无奈，什么都没有，那到底 Eureka Client端的组件是怎么加载的呢？ 



## 通过 spring.factories

我们既然是client端，就来查看一下其包下的spring.factories。

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration

org.springframework.cloud.bootstrap.BootstrapConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceBootstrapConfiguration
```

我们一个一个来看看这些自动装配。

#### EurekaClientConfigServerAutoConfiguration

我们来看一下第一个自动装配,这个自动装配类，装配的目的是引导外部化配置当恰好有一个Eureka实例。

```java
@Configuration
@EnableConfigurationProperties
//是否存在这些类的实例如果有一个不存在则不初始化
//这里重点关注一下ConfigServerProperties
//他是我们需要引用spring cloud config相关依赖才有的
//因此后续可能学到相关内容会有理解
@ConditionalOnClass({ EurekaInstanceConfigBean.class, EurekaClient.class,
      ConfigServerProperties.class })
public class EurekaClientConfigServerAutoConfiguration {

   @Autowired(required = false)
   private EurekaInstanceConfig instance;

   @Autowired(required = false)
   private ConfigServerProperties server;

   @PostConstruct
   public void init() {
      if (this.instance == null || this.server == null) {
         return;
      }
      String prefix = this.server.getPrefix();
      if (StringUtils.hasText(prefix) && !StringUtils
            .hasText(this.instance.getMetadataMap().get("configPath"))) {
         this.instance.getMetadataMap().put("configPath", prefix);
      }
   }

}
```

由于我们没有引用spring-cloud-config相关的依赖，所以这个自动装配不注册。



#### EurekaDiscoveryClientConfigServiceAutoConfiguration

这个自动装配是 引导一个配置服务端去发现一个配置服务。

```java
@ConditionalOnBean({ EurekaDiscoveryClientConfiguration.class })
//如果我们在外部化配置中，配置了
//spring.cloud.config.discovery.enabled 就自动装配 否则不装配
@ConditionalOnProperty(value = "spring.cloud.config.discovery.enabled", matchIfMissing = false)
public class EurekaDiscoveryClientConfigServiceAutoConfiguration {

   @Autowired
   private ConfigurableApplicationContext context;

   @PostConstruct
   public void init() {
      if (this.context.getParent() != null) {
         if (this.context.getBeanNamesForType(EurekaClient.class).length > 0
               && this.context.getParent()
                     .getBeanNamesForType(EurekaClient.class).length > 0) {
            // If the parent has a EurekaClient as well it should be shutdown, so the
            // local one can register accurate instance info
            this.context.getParent().getBean(EurekaClient.class).shutdown();
         }
      }
   }

}
```



由于在这里我们也没有过配置`spring.cloud.config.discovery.enabled`相关外部化内容，所以这个自动装配也不装配。



#### org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration

这个才是我们真正所需要找的自动装配的类。由于代码冗长，所以还是分开来看。先从注解入手。

##### 注解--装配前的准备

```java
@Configuration
@EnableConfigurationProperties
//是否存在该类在路径中
@ConditionalOnClass(EurekaClientConfig.class)
//引入过滤器 jersey 1,2 相关的过滤器
@Import(DiscoveryClientOptionalArgsConfiguration.class)
//是否存在实例Marker
@ConditionalOnBean(EurekaDiscoveryClientConfiguration.Marker.class)
//外部化配置是否配置了 eureka.client.enabled 属性
//如果没有 也可以实例化
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
//模式注解 也是判断外部化配置是否配置了 eureka.client.enabled 属性
@ConditionalOnDiscoveryEnabled
//在这些类装配之前进行装配
@AutoConfigureBefore({ NoopDiscoveryClientAutoConfiguration.class,
      CommonsClientAutoConfiguration.class, ServiceRegistryAutoConfiguration.class })
//在这些类装配之后进行装配
@AutoConfigureAfter(name = {
      //配置监听应用程序的事件以及刷新相关配置
      "org.springframework.cloud.autoconfigure.RefreshAutoConfiguration",
       //这个负责启动实现服务注册发现功能的配置
      "org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration",
     //自动服务注册的相关配置 
  "org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration" })
public class EurekaClientAutoConfiguration {
    //代码省略
}
```





##### 代码--服务注册发起

```java
public class EurekaClientAutoConfiguration {

//省略部分代码

   @Bean
   public DiscoveryClient discoveryClient(EurekaClient client,
         EurekaClientConfig clientConfig) {
      return new EurekaDiscoveryClient(client, clientConfig);
   }
}
//省略部分代码
}
```

我们知道，注册服务肯定是通过客户端来完成的，那么在spring cloud中，上述代码就是实例化DiscoveryClient对象的方法，但是我们也可以看到，这简简单单的创建对象，是如何完成注册的呢。

首先我们先来看一下`EurekaDiscoveryClient`这个类。

```java
package org.springframework.cloud.netflix.eureka;
/**
 * A {@link DiscoveryClient} implementation for Eureka.
 *
 * @author Spencer Gibb
 * @author Tim Ysewyn
 */
public class EurekaDiscoveryClient implements DiscoveryClient {

   /**
    * Client description {@link String}.
    */
   public static final String DESCRIPTION = "Spring Cloud Eureka Discovery Client";
  //netflix 开源包中的 Client
   private final EurekaClient eurekaClient;
   //netflix 开源包中的 EurekaClientConfig
   private final EurekaClientConfig clientConfig;

   //构造函数
   public EurekaDiscoveryClient(EurekaClient eurekaClient,
         EurekaClientConfig clientConfig) {
      this.clientConfig = clientConfig;
      this.eurekaClient = eurekaClient;
   }
}
```

我故意把这个类的包引用给copy了出来`org.springframework.cloud.netflix.eureka`，就是要大家知道，这个类其实是spring调用netflix开源包中的代理类。



那么真正的实例化EurekaClient的地方在哪呢？我们往下看。可以看到两个内部类。

###### 1.EurekaClientConfiguration

```java
@Configuration
//如果缺失 org.springframework.cloud.context.scope.refresh.RefreshScope
//则执行后续操作
@ConditionalOnMissingRefreshScope
protected static class EurekaClientConfiguration {
 //省略部分代码
   @Bean(destroyMethod = "shutdown")
   @ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
   public EurekaClient eurekaClient(ApplicationInfoManager manager,
         EurekaClientConfig config) {
      return new CloudEurekaClient(manager, config, this.optionalArgs,
            this.context);
   }
   //省略部分代码
   }
```

我们可以看到在这个类中，有实例化EurekaClient的方法，但是到底是不是这个类呢。答案是否定的 因为该类方法中有一个条件约束注解`@ConditionalOnMissingRefreshScope`。具体的约束条件在注释中已经给出。这就要回到我们刚刚的**注解--装配前的准备**那段了，有一个注解是 `@AutoConfigureAfter`其中第一个条件类就是`org.springframework.cloud.autoconfigure.RefreshAutoConfiguration` 而在这个条件类中，我们就可以看到`RefreshScope`的实例化。

`org.springframework.cloud.autoconfigure.RefreshAutoConfiguration`

```java
@Configuration
@ConditionalOnClass(RefreshScope.class)
@ConditionalOnProperty(name = RefreshAutoConfiguration.REFRESH_SCOPE_ENABLED, matchIfMissing = true)
@AutoConfigureBefore(HibernateJpaAutoConfiguration.class)
public class RefreshAutoConfiguration {
//省略部分代码
	@Bean
	@ConditionalOnMissingBean(RefreshScope.class)
	public static RefreshScope refreshScope() {
		return new RefreshScope();
	}
	//省略部分代码
	}
```

因此，上述这个类`EurekaClientConfiguration`不执行。



###### 2.RefreshableEurekaClientConfiguration

通过刚刚上面那个类的初始化流程，相信大家看到这个类就知道肯定是这个类去装载的`EurekaClient`

```java
@Configuration
@ConditionalOnRefreshScope
protected static class RefreshableEurekaClientConfiguration {
   //省略部分代码
    //实例摧毁前调用 shutdown 方法
   @Bean(destroyMethod = "shutdown")
   @ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
    //每次加载都刷新一个实例
   @org.springframework.cloud.context.config.annotation.RefreshScope
    //懒加载
   @Lazy
   public EurekaClient eurekaClient(ApplicationInfoManager manager,
         EurekaClientConfig config, EurekaInstanceConfig instance,
         @Autowired(required = false) HealthCheckHandler healthCheckHandler) {
     //初始化Eureka Server 和其他准备发现的组件所需要的信息
      ApplicationInfoManager appManager;
       //获取代理
      if (AopUtils.isAopProxy(manager)) {
         appManager = ProxyUtils.getTargetObject(manager);
      }
      else {
         appManager = manager;
      }
       //实例化 DiscoveryClient的子类 CloudEurekaClient
      CloudEurekaClient cloudEurekaClient = new CloudEurekaClient(appManager,
            config, this.optionalArgs, this.context);
      cloudEurekaClient.registerHealthCheck(healthCheckHandler);
      return cloudEurekaClient;
   }
}
```

找了这么久终于找到了 实例化CloudEurekaClient的地方，我们来看一下CloudEurekaClient的源码。

```java
public class CloudEurekaClient extends DiscoveryClient {
    //省略部分代码
    public CloudEurekaClient(ApplicationInfoManager applicationInfoManager,
          EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs<?> args,
          ApplicationEventPublisher publisher) {
        //调用父类的初始化方法
       super(applicationInfoManager, config, args);
       this.applicationInfoManager = applicationInfoManager;
       this.publisher = publisher;
       this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class,
             "eurekaTransport");
       ReflectionUtils.makeAccessible(this.eurekaTransportField);
    }
    //省略部分代码
}
```



我们可以看到，构造函数中调用了父类的构造方法，我们看一下父类构造函数。

```java
@Inject
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                Provider<BackupRegistry> backupRegistryProvider) {
  //忽略部分代码 客户端相关配置读取以及配置
    //是否从客户端获取注册信息
    if (config.shouldFetchRegistry()) {
        this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
    } else {
        this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
    }
    //是否使用eureka注册信息
    if (config.shouldRegisterWithEureka()) {
        this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
    } else {
        this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
    }

    logger.info("Initializing Eureka in region {}", clientConfig.getRegion());

   //忽略部分代码
    try {
        // default size of 2 - 1 each for heartbeat and cacheRefresh
        //定义一个定时线程池，大小为2
        //目的是包含一个 刷新注册信息线程和一个心跳机制线程
        scheduler = Executors.newScheduledThreadPool(2,
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-%d")
                        .setDaemon(true)
                        .build());

        //心跳机制线程
        heartbeatExecutor = new ThreadPoolExecutor(
                1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff

        //注册信息刷新线程
        cacheRefreshExecutor = new ThreadPoolExecutor(
                1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff

//省略部分代码

    // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
   //初始化定时任务 包括服务的注册信息获取，心跳机制等     
    initScheduledTasks();

    try {
        Monitors.registerObject(this);
    } catch (Throwable e) {
        logger.warn("Cannot register timers", e);
    }

    // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
    // to work with DI'd DiscoveryClient
    DiscoveryManager.getInstance().setDiscoveryClient(this);
    DiscoveryManager.getInstance().setEurekaClientConfig(config);

    initTimestampMs = System.currentTimeMillis();
    logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
            initTimestampMs, this.getApplications().size());
}
```



我们通过源码可以看到 设置了定时任务线程池以及两个后台线程池（获取注册信息，心跳机制）。通过

`initScheduledTasks()`开启任务。我们来看一下详细的代码。

```java
/**
 * Initializes all scheduled tasks.
 */
private void initScheduledTasks() {
    if (clientConfig.shouldFetchRegistry()) {
        // registry cache refresh timer
        //刷新获取注册信息的时间
        int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
        //最大等待超时时长
        int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
        //定时获取刷新注册信息任务
        scheduler.schedule(
                new TimedSupervisorTask(
                        "cacheRefresh",
                        scheduler,
                        cacheRefreshExecutor,
                        registryFetchIntervalSeconds,
                        TimeUnit.SECONDS,
                        expBackOffBound,
                        new CacheRefreshThread()
                ),
                registryFetchIntervalSeconds, TimeUnit.SECONDS);
    }

    if (clientConfig.shouldRegisterWithEureka()) {
        int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
        int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
        logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

        // Heartbeat timer
        //心跳机制定时
        scheduler.schedule(
                new TimedSupervisorTask(
                        "heartbeat",
                        scheduler,
                        heartbeatExecutor,
                        renewalIntervalInSecs,
                        TimeUnit.SECONDS,
                        expBackOffBound,
                        new HeartbeatThread()
                ),
                renewalIntervalInSecs, TimeUnit.SECONDS);

        // InstanceInfo replicator
        //将客户端的信息注册
        instanceInfoReplicator = new InstanceInfoReplicator(
                this,
                instanceInfo,
                clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                2); // burstSize

        statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
            @Override
            public String getId() {
                return "statusChangeListener";
            }

            @Override
            public void notify(StatusChangeEvent statusChangeEvent) {
                if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                        InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                    // log at warn level if DOWN was involved
                    logger.warn("Saw local status change event {}", statusChangeEvent);
                } else {
                    logger.info("Saw local status change event {}", statusChangeEvent);
                }
                instanceInfoReplicator.onDemandUpdate();
            }
        };

        if (clientConfig.shouldOnDemandUpdateStatusChange()) {
            applicationInfoManager.registerStatusChangeListener(statusChangeListener);
        }

       //注册服务
  instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }
}
```

这里简单的讲述一下注册服务这个方法，由于InstanceInfoReplicator实现了Runnable接口。因此当调用start方法时。

```java
public void start(int initialDelayMs) {
    if (started.compareAndSet(false, true)) {
        instanceInfo.setIsDirty();  // for initial register
        Future next = scheduler.schedule(this, initialDelayMs, TimeUnit.SECONDS);
        scheduledPeriodicRef.set(next);
    }
}
```



定时任务会执行自身的类的run()方法，也就是真正执行注册的方法。

```java
public void run() {
    try {
        discoveryClient.refreshInstanceInfo();

        Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
        if (dirtyTimestamp != null) {
            discoveryClient.register();
            instanceInfo.unsetIsDirty(dirtyTimestamp);
        }
    } catch (Throwable t) {
        logger.warn("There was a problem with the instance info replicator", t);
    } finally {
        Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
        scheduledPeriodicRef.set(next);
    }
}
```

具体不在深入，感兴趣的同学可以自己探索。



## 小结

​	随着慢慢了解Spring Cloud Eureka 源码，我们对于Eureka 注册中心机制也有了更深的理解