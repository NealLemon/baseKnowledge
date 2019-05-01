# Spring Cloud系列--Feign整合(二)

之前讲了简单的Feign整合，接下来我们趁热打铁，把Ribbon和Hystrix整合进来。

## Feign和Ribbon

在学习Feign的时候，我们都了解到 Feign的客户端负载均衡是通过Spring Cloud Ribbon 实现的。具体的实现和源码，以后会有介绍，在这里就简单的介绍负载均衡组件装配的关系。

我们通过查找`spring-cloud-openfeign-core`源码包中的 `/META-INF/spring.factories`

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.openfeign.ribbon.FeignRibbonClientAutoConfiguration,\
org.springframework.cloud.openfeign.FeignAutoConfiguration,\
org.springframework.cloud.openfeign.encoding.FeignAcceptGzipEncodingAutoConfiguration,\
org.springframework.cloud.openfeign.encoding.FeignContentGzipEncodingAutoConfiguration
```



我们打开第一个要自动装配文件。

```java
//类路径下是否存在负载均衡接口
@ConditionalOnClass({ ILoadBalancer.class, Feign.class })  
@Configuration
@AutoConfigureBefore(FeignAutoConfiguration.class)
@EnableConfigurationProperties({ FeignHttpClientProperties.class })
//引入其他配置组件
@Import({ HttpClientFeignLoadBalancedConfiguration.class,
      OkHttpFeignLoadBalancedConfiguration.class,
      DefaultFeignLoadBalancedConfiguration.class })
public class FeignRibbonClientAutoConfiguration {

   //创建所需的工厂组件
   @Bean
   @Primary
   @ConditionalOnMissingBean
   @ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
   public CachingSpringLoadBalancerFactory cachingLBClientFactory(
         SpringClientFactory factory) {
      return new CachingSpringLoadBalancerFactory(factory);
   }
  //创建所需的工厂组件
   @Bean
   @Primary
   @ConditionalOnMissingBean
   @ConditionalOnClass(name = "org.springframework.retry.support.RetryTemplate")
   public CachingSpringLoadBalancerFactory retryabeCachingLBClientFactory(
         SpringClientFactory factory, LoadBalancedRetryFactory retryFactory) {
      return new CachingSpringLoadBalancerFactory(factory, retryFactory);
   }

   @Bean
   @ConditionalOnMissingBean
   public Request.Options feignRequestOptions() {
      return LoadBalancerFeignClient.DEFAULT_OPTIONS;
   }

}
```



重点是 在注解`@import`中引入的 `DefaultFeignLoadBalancedConfiguration.class ` 

```java
/**
 * @author Spencer Gibb
 */
@Configuration
class DefaultFeignLoadBalancedConfiguration {

   @Bean
   @ConditionalOnMissingBean
   public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
         SpringClientFactory clientFactory) {
      return new LoadBalancerFeignClient(new Client.Default(null, null), cachingFactory,
            clientFactory);
   }

}
```

从这我们可以看出，创建了一个`LoadBalancerFeignClient`组件。

我们再来深入的看一下这个类。

```java
/**
 * @author Dave Syer
 *
 */
public class LoadBalancerFeignClient implements Client {
    //省略部分代码
    	@Override
	public Response execute(Request request, Request.Options options) throws IOException {
		try {
			URI asUri = URI.create(request.url());
			String clientName = asUri.getHost();
			URI uriWithoutHost = cleanUrl(request.url(), clientName);
            //获得RibbonRequest
			FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
					this.delegate, request, uriWithoutHost);

			IClientConfig requestConfig = getClientConfig(options, clientName);
            //进行使用RibbonRequest负载均衡请求调用
			return lbClient(clientName)
					.executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
		}
		catch (ClientException e) {
			IOException io = findIOException(e);
			if (io != null) {
				throw io;
			}
			throw new RuntimeException(e);
		}
	}
//...
}
```



到这里 应该也算是对Feign和Ribbon之间的关系有所了解。具体的两者之间的关系，有时间，会进一步深入的研究。



## 整合Hystrix

整合Hystrix其实不需要改变之前的 注册中心（`spring-cloud-test-register`）。只需要修改客户端(`myribbon`)和服务提供方(`provider`)就可以了。

#### 客户端(`myribbon`)

##### 添加`StudentServiceFallBack` 熔断降级处理类

```java
/**
 * @ClassName StudentServiceFallBack
 * @Description 熔断回调函数接口
 * @Author Neal
 * @Date 2019/4/29 21:39
 * @Version 1.0
 */
@Component
public class StudentServiceFallBack implements StudentService {
    @Override
    public String getAllStudent() {
        return "getAllStudent()  超时降级处理";
    }

    @Override
    public String saveStudent(Student student) {
        return "saveStudent() 超时降级处理";
    }
}
```

我们可以看到，这个类很简单，首先追加 `@Component`注解，然后实现`StudentService`接口。将实现的方法改为降级处理的逻辑即可。



##### Feign API接口添加降级回调

```java
/**
 * @ClassName StudentService
 * @Description feign API接口
 * @Author Neal
 * @Date 2019/4/29 14:36
 * @Version 1.0
 */

@FeignClient(name = "RIBBON-SERVICE",fallback = StudentServiceFallBack.class)  //服务端ID
public interface StudentService {

    /**
     * 获取所有学生列表
     * @return
     */
    @GetMapping(value = "/myfeign/student")
    String getAllStudent();

    /**
     * 添加学生
     * @param student
     * @return
     */
    @PostMapping(value = "/myfeign/student")
    String saveStudent(@RequestBody Student student);
}
```



##### 启动类上添加	`@EnableHystrix`注解

```java
@EnableEurekaClient
@SpringBootApplication
@EnableHystrix
@EnableFeignClients(clients = StudentService.class)  //开启
public class MyribbonApplication {

    public static void main(String[] args) {
        SpringApplication.run(MyribbonApplication.class, args);
    }

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
```





好了，现在代码就这么多了。是不是很简单。我们接着来操作一下外部化配置。

`application-consumer.properties`

```properties
spring.application.name=ribbon-consumer
server.port=1114
eureka.client.service-url.defaultZone=http://localhost:1111/eureka/
#允许feign使用 hystrix
feign.hystrix.enabled=true
#降级默认超时时间
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=50
```



#### 服务提供方(`provider`)

只需要修改API调用方法，增加模拟延迟处理逻辑即可。同时添加了 外部化配置 的端口。方便查看负载均衡。

```java
/**
 * @ClassName StudentController
 * @Description TODO
 * @Author Neal
 * @Date 2019/4/29 14:58
 * @Version 1.0
 */
@RestController
public class StudentController implements  StudentService{

    @Autowired
    @Qualifier("iStudentServiceImp")
    StudentService studentService;

    /**
     * 默认服务端口，用来观察负载均衡
     */
    @Value("${server.port}")
    private String port;

    /**
     * 随机数生成类
     */
    private static Random random = new Random();


    /**
     * 增加延迟逻辑处理
     * @return
     */
    @Override
    public String getAllStudent() {
        long executeTime = random.nextInt(100);
        System.out.println("execute Time : " + executeTime);
        try {
            TimeUnit.MILLISECONDS.sleep(executeTime);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        return studentService.getAllStudent() + "port:" + port;
    }

    @Override
    public String saveStudent(Student student) {
        return studentService.saveStudent(student);
    }

    /**
     * 熔断策略的回调方法
     * @return
     */
    public String fallBackHelloWorld() {
        return "@HystrixCommand : Time out 100 ms";
    }
}
```





这里的服务端口我在properties文件中设置的。由于要实现两个服务端，所以分了两个properties文件，如何启动，可以参照之前的文章。

`application-provider1.properties`

```properties
spring.application.name=ribbon-service
#端口
server.port=1112
eureka.client.service-url.defaultZone=http://localhost:1111/eureka/
```



`application-provider2.properties`

```properties
spring.application.name=ribbon-service
#端口
server.port=1113
eureka.client.service-url.defaultZone=http://localhost:1111/eureka/
```



## 运行

基础的代码就这么多，下面让我们跑一下程序来看看是否实现了 负载均衡和熔断降级吧。

启动程序，顺序为:

`spring-cloud-test-register`-> `provider`(`provider1`和`provider2`) -> `myribbon`



首先查看注册中心，是否注册了所有服务。

注册.png



我们可以看到 服务端的两个服务以及消费者已经都注册到了注册中心上。

我们可以先简单插入一条数据（跟上一篇一样，这里我就不重复操作了），然后请求查看所有学生的API接口，看看是否有熔断降级和负载均衡。

http://localhost:1114/myfeign/student



r1.png

p1.png

r2.png

p2.png



我们可以看到 provider1和provider2 实现了负载均衡处理，并且 请求处理的时间都不超过50毫秒，所以不会有降级操作。	



#### 熔断降级

h1.png

p3.png



我们也可以看到，由于处理时间超过了50毫秒，客户端自行进行了熔断降级处理。



## 总结

  Feign的基本内容已经介绍完了。如果真的感兴趣或者想知道更细致的操作，可以参考官方的文档，比网上其他的博客靠谱的多。