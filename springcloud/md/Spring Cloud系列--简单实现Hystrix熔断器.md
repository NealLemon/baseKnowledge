# Spring Cloud系列--简单实现Hystrix熔断器

好久没更新了，因为之前有点感情问题，颓废了一段时间，日子还得往下过，技术还得继续学啊。这篇文章是我自己看完书和相关视频后，自己基于之前的环境搭建简单的Hystrix熔断器。

## 概要

首先Hystrix熔断器可以是服务端也可以是客户端的，因此我会分别实现客户端和服务端的熔断器。



## 基础环境搭建

首先我们必须要有3个项目。具体搭建可以参考[Spring Cloud系列--简单实现Ribbon负载均衡](https://www.jianshu.com/p/e828b63a5cd8) 

- spring-cloud-test-register
- provider
- myribbon



## 服务端实现Hystrix熔断器

1. 我们这先修改`provider`这里的项目代码。

```java
/**
 * @ClassName HelloWorldController
 * @Description TODO
 * @Author Neal
 * @Date 2019/3/25 10:49
 * @Version 1.0
 */
@RestController
public class HelloWorldController {

    @Value("${server.port}")
    private String port;

    /**
     * 随机数生成类
     */
    private static Random random = new Random();

    /**
     * 基于服务端的熔断策略实现
     * @return
     * @throws InterruptedException
     */
    @HystrixCommand(
            commandProperties = {
      //设置单线程请求的超时时间，如果超时，回调方法    
                @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value="100")
            },
            fallbackMethod = "fallBackHelloWorld"
    )
    @GetMapping("/hello/hystrix")
    public String hystrixHelloWorld() throws InterruptedException {
        long executeTime = random.nextInt(200);
        System.out.println("execute Time : " + executeTime);
        TimeUnit.MILLISECONDS.sleep(executeTime);
        return "Hello World from port:" + port;
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



 2.修改myribbon项目代码。

```java
@RestController
public class HelloConsumerController {

    @Autowired
    RestTemplate restTemplate;


    /**
     * 服务端熔断请求的方法
     * @return
     */
    @GetMapping("/helloworld/hystrix")
    public String hystrixHelloWorld() {
        return restTemplate.getForEntity("http://RIBBON-SERVICE/hello/hystrix",String.class).getBody();
    }
}
```



代码梳理完之后，我们顺序启动程序。spring-cloud-test-register -> provider -> myribbon



通过浏览器直接进行访问操作。

超时时熔断结果

p1.png

p2.png



未超时时的结果

p3.png

p4.png



## 客户端熔断实现

1.我们这先修改`provider`这里的项目代码。

```java
@RestController
public class HelloWorldController {

    @Value("${server.port}")
    private String port;

    /**
     * 随机数生成类
     */
    private static Random random = new Random();

    /**
     * 基于客户端的熔断方法实现
     * @return
     * @throws InterruptedException
     */
    @GetMapping("/hello")
    public String helloWorld() throws InterruptedException {

        long executeTime = random.nextInt(200);
        System.out.println("execute Time : " + executeTime);
        TimeUnit.MILLISECONDS.sleep(executeTime);

        return "Hello World from port:" + port;
    }
}
```



2.修改myribbon项目代码。

a.首先新建`HelloWorldCommand`

```java
/**
 * @ClassName HelloWorldCommand
 * @Description 客户端熔断器类
 * @Author Neal
 * @Date 2019/4/18 10:54
 * @Version 1.0
 */
public class HelloWorldCommand extends HystrixCommand<String> {

    private final String url;

    private  final RestTemplate restTemplate;

    /**
     * 构造函数，必须！
     * @param group
     * @param executionIsolationThreadTimeoutInMilliseconds
     * @param url
     * @param restTemplate
     */
    public HelloWorldCommand(HystrixCommandGroupKey group, int executionIsolationThreadTimeoutInMilliseconds, String url, RestTemplate restTemplate) {
        super(group, executionIsolationThreadTimeoutInMilliseconds);
        this.url = url;
        this.restTemplate = restTemplate;
    }

    /**
     * 执行请求操作
     * @return
     */
    @Override
    protected String run() {
        return restTemplate.getForEntity(url,String.class).getBody();
    }


    /**
     * 执行回调方法
     * @return
     */
    @Override
    protected String getFallback() {
        return "HelloWorldCommand time is out 100 ms";
    }
}
```



b.向controller层新增方法。

```java
@RestController
public class HelloConsumerController {

    @Autowired
    RestTemplate restTemplate;
    /**
    * 客户端熔断请求方法
    */
    @GetMapping("/helloworld")
    public String helloWorld() {
        return new HelloWorldCommand(HystrixCommandGroupKey.Factory.asKey("Ribbon Client"),
                100,
                "http://RIBBON-SERVICE/hello",restTemplate).execute();
    }
}
```

代码梳理完之后，我们顺序启动程序。spring-cloud-test-register -> provider -> myribbon



通过浏览器直接进行访问操作。

超时时熔断结果

p5.png

p6.png



未超时时的结果

p7.png

p8.png



## 总结

简单的Spring cloud Hystrix 熔断已经搭建完了，虽然Hystrix  不是最优选择的熔断器，但是可以从熔断思维上理解这个组件，以后面对各种熔断器，也知道具体原理了。