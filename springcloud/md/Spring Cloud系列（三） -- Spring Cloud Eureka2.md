# Spring Cloud系列（三）-- Spring Cloud Eureka源码(二)

上一篇介绍了Eureka Server端初始化时的自动装配过程`EurekaServerAutoConfiguration` ,现在我们来看一下server端的启动源码。

## 导入启动类

在`EurekaServerAutoConfiguration`  中，我们看到在自动装配的时候导入了一个类，那就是`EurekaServerInitializerConfiguration` 这个类就是Eureka Server端的启动调用类。

#### @import(EurekaServerInitializerConfiguration.class)

在我们分析`EurekaServerAutoConfiguration`  源码的时候有这个这么一个注册bean。

```java
   @Bean
   public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
         EurekaServerContext serverContext) {
      return new EurekaServerBootstrap(this.applicationInfoManager,
            this.eurekaClientConfig, this.eurekaServerConfig, registry,
            serverContext);
   }
```

这个注册Bean 在导入类中就使用到了。我们来看一下这个类`EurekaServerInitializerConfiguration`的源码。

```java
@Configuration
public class EurekaServerInitializerConfiguration
      implements ServletContextAware, SmartLifecycle, Ordered {

 //省略部分源码

   @Autowired
   private EurekaServerConfig eurekaServerConfig;

   private ServletContext servletContext;

   @Autowired
   private ApplicationContext applicationContext;

   @Autowired
   private EurekaServerBootstrap eurekaServerBootstrap;

  //省略部分源码
   //新启动一个线程来启动 eureka Server
   @Override
   public void start() {
      new Thread(new Runnable() {
         @Override
         public void run() {
            try {
               //初始化eurekaServer 上下文 并启动 eurekaServer
               eurekaServerBootstrap.contextInitialized(
                     EurekaServerInitializerConfiguration.this.servletContext);
               log.info("Started Eureka Server");
				//发布可以注册时间
               publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
               EurekaServerInitializerConfiguration.this.running = true;
                //发布启动事件
               publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
            }
            catch (Exception ex) {
               // Help!
               log.error("Could not initialize Eureka servlet context", ex);
            }
         }
      }).start();
   }
 //省略部分源码

}
```

我们可以看到，其实这段源码中没有多少营养的地方，主要就是另启动一个线程，去操作`eurekaServerBootstrap`的启动类。所以我们需要仔细看一下 `eurekaServerBootstrap`的源码。



## EurekaServerBootstrap

我们截取重要的部分来看一下就可以了。

```java
/**
 * @author Spencer Gibb
 */
public class EurekaServerBootstrap {

  //省略部分源码
  //这里就是 在EurekaServerInitializerConfiguration中调用的启动方法
   public void contextInitialized(ServletContext context) {
      try {
          //初始化 环境参数
         initEurekaEnvironment();
          //初始化并且配置Eureka 上下文
         initEurekaServerContext();

         context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
      }
      catch (Throwable e) {
         log.error("Cannot bootstrap eureka server :", e);
         throw new RuntimeException("Cannot bootstrap eureka server :", e);
      }
   }

    //销毁上下文方法
   public void contextDestroyed(ServletContext context) {
      try {
         log.info("Shutting down Eureka Server..");
         context.removeAttribute(EurekaServerContext.class.getName());

         destroyEurekaServerContext();
         destroyEurekaEnvironment();

      }
      catch (Throwable e) {
         log.error("Error shutting down eureka", e);
      }
      log.info("Eureka Service is now shutdown...");
   }

    //初始化 环境参数
   protected void initEurekaEnvironment() throws Exception {
      log.info("Setting the eureka configuration..");

       //设置配置文件的数据中心 因为没有学到相关内容，具体作用不知，如果后续
       //有学到会更新
      String dataCenter = ConfigurationManager.getConfigInstance()
            .getString(EUREKA_DATACENTER);
      if (dataCenter == null) {
         log.info(
               "Eureka data center value eureka.datacenter is not set, defaulting to default");
         ConfigurationManager.getConfigInstance()
               .setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, DEFAULT);
      }
      else {
         ConfigurationManager.getConfigInstance()
               .setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, dataCenter);
      }
       //获取配置文件的环境变量
      String environment = ConfigurationManager.getConfigInstance()
            .getString(EUREKA_ENVIRONMENT);
       //设置默认的环境变量
      if (environment == null) {
         ConfigurationManager.getConfigInstance()
               .setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, TEST);
         log.info(
               "Eureka environment value eureka.environment is not set, defaulting to test");
      }
      else {
         ConfigurationManager.getConfigInstance()
               .setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, environment);
      }
   }

   //初始化上下文
   protected void initEurekaServerContext() throws Exception {
      // 为了向后兼容，文本转换工具 方便JAVA对象与文本格式数据的相互转换
      JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
            XStream.PRIORITY_VERY_HIGH);
      XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
            XStream.PRIORITY_VERY_HIGH);

       //是否是 AWS云服务
      if (isAws(this.applicationInfoManager.getInfo())) {
         this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig,
               this.eurekaClientConfig, this.registry, this.applicationInfoManager);
         this.awsBinder.start();
      }

      //初始化静态上下文持有
      EurekaServerContexHolder.initialize(this.serverContext);

      log.info("Initialized server context");

      // 从相邻节点copy注册表   具体内容下面有接受
      int registryCount = this.registry.syncUp();
       //开启注册通道 具体内容下面有解释
      this.registry.openForTraffic(this.applicationInfoManager, registryCount);

      // 注册Eureka监控的所有统计信息的映射。
      EurekaMonitors.registerAllStats();
   }

   /**
    * Server context shutdown hook. Override for custom logic
    * @throws Exception - calling {@link AwsBinder#shutdown()} or
    * {@link EurekaServerContext#shutdown()} may result in an exception
    */
   protected void destroyEurekaServerContext() throws Exception {
      EurekaMonitors.shutdown();
      if (this.awsBinder != null) {
         this.awsBinder.shutdown();
      }
      if (this.serverContext != null) {
         this.serverContext.shutdown();
      }
   }

   /**
    * Users can override to clean up the environment themselves.
    * @throws Exception - shutting down Eureka servers may result in an exception
    */
   protected void destroyEurekaEnvironment() throws Exception {
   }

   protected boolean isAws(InstanceInfo selfInstanceInfo) {
      boolean result = DataCenterInfo.Name.Amazon == selfInstanceInfo
            .getDataCenterInfo().getName();
      log.info("isAws returned " + result);
      return result;
   }

}
```



#### 重点方法补充

##### this.registry.syncUp()

我们在`org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap#initEurekaServerContext`方法中 有一段代码,这段代码的功能是同步注册列表

```java
int registryCount = this.registry.syncUp();
```

我们来看一下这个方法的源码，如果不知道这个`registry`注册对象从哪来或者是什么 可以看上一篇内容。这里直接给出类以及方法的具体代码。

`com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#syncUp`

```java
/**
 * Populates the registry information from a peer eureka node. This
 * operation fails over to other nodes until the list is exhausted if the
 * communication fails.
 */
//从一个节点获取注册列表填充到本地节点
@Override
public int syncUp() {
    // Copy entire entry from neighboring DS node
    int count = 0;
    //尝试同步一定的次数，如果超过一定的同步次数，则放弃
    for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
        if (i > 0) {
            try {
                Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
            } catch (InterruptedException e) {
                logger.warn("Interrupted during registry transfer..");
                break;
            }
        }
        
        Applications apps = eurekaClient.getApplications();
        //获取注册信息，并同步到本地
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                try {
                    if (isRegisterable(instance)) {
                        register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                        count++;
                    }
                } catch (Throwable t) {
                    logger.error("During DS init copy", t);
                }
            }
        }
    }
    return count;
}
```



##### this.registry.openForTraffic(this.applicationInfoManager, registryCount);

我们在`org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap#initEurekaServerContext`方法中 有另外一段代码，这段代码的意义是开启注册通道

```java
this.registry.openForTraffic(this.applicationInfoManager, registryCount);
```

我们来看一下这段方法的源码。

```java
@Override
public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
    //同步之后，获取到的注册数据个数
    this.expectedNumberOfClientsSendingRenews = count;
    // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
    //每30秒 重新更新一次
    updateRenewsPerMinThreshold();
    logger.info("Got {} instances from neighboring DS node", count);
    logger.info("Renew threshold is: {}", numberOfRenewsPerMinThreshold);
    this.startupTime = System.currentTimeMillis();
    if (count > 0) {
        this.peerInstancesTransferEmptyOnStartup = false;
    }
    DataCenterInfo.Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
    boolean isAws = Name.Amazon == selfName;
    if (isAws && serverConfig.shouldPrimeAwsReplicaConnections()) {
        logger.info("Priming AWS connections for all replicas..");
        primeAwsReplicas(applicationInfoManager);
    }
    logger.info("Changing status to UP");
    //设置实例状态，表示已经准备好接收流量，同时通知所有监听该状态的改变事件
    applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
    super.postInit();
}
```





## 总结

Eureka Server 端的启动过程的源码基本已经介绍完了，当然这只是源码的大体介绍，在自动装配过程中，每个Bean的注册，其实都非常有意义去探讨。由于Eureka Server 相关的Bean 是在太多了，真的看不过来。感兴趣的小伙伴可以自行了解一下。