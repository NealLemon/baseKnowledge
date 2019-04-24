# Spring Cloud系列--Hystrix源码（一）

首先在了解Hystrix源码之前，都必须了解一下[RxJava](https://github.com/ReactiveX/RxJava)。 引用官方的一句话来解释一下RxJava到底是干嘛的。

> RxJava is a Java VM implementation of [Reactive Extensions](http://reactivex.io): a library for composing asynchronous and event-based programs by using observable sequences.

来自谷歌翻译：RxJava是Reactive Extensions的Java VM实现：一个使用可观察序列组成异步和基于事件的程序的库。

其实就是一个观察者模式的一种封装和优化的实现。



## 从@EnableHystrix开始我们的源码之旅

  在上一篇文章中，我们在provider项目中的启动类中增加了`@EnableHystrix`注解。

```java
@EnableDiscoveryClient
@EnableHystrix
@SpringBootApplication
public class ProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProviderApplication.class, args);
    }

}
```



那么我们就从这个注解入手，来挖掘一下spring-cloud-Hystrix源码。

#### @EnableHystrix

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@EnableCircuitBreaker
public @interface EnableHystrix {
}
```

我们可以看到 这个注解之上还有个模式注解，其实细心的同学就可以通过名字发现这个注解的真实含义。

就是打开Hystrix熔断器的一个注解开关。那么真正的实现在哪呢，我们接着往下看。



#### @EnableCircuitBreaker

我们进入到`@EnableCircuitBreaker`注解中。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(EnableCircuitBreakerImportSelector.class)
public @interface EnableCircuitBreaker {

}
```

我想这就是我们想要的真正打开熔断器的注解了。在之前spring-boot中我们讲到过 `@import`模式注解的作用，我们这里就不多做解释了。我们直接看`EnableCircuitBreakerImportSelector.class`。



#### @EnableCircuitBreakerImportSelector

```java
@Order(Ordered.LOWEST_PRECEDENCE - 100)
public class EnableCircuitBreakerImportSelector
      extends SpringFactoryImportSelector<EnableCircuitBreaker> {

    /**
    *  根据外部化匹配 spring.cloud.circuit.breaker.enabled 
    *  控制是否开启熔断器的判断方法
    */
   @Override
   protected boolean isEnabled() {
      return getEnvironment().getProperty("spring.cloud.circuit.breaker.enabled",
            Boolean.class, Boolean.TRUE);
   }

}
```



这里`EnableCircuitBreakerImportSelector`我们继承了`SpringFactoryImportSelector`类。这个类的作用是 由通用类型T(这里指`EnableCircuitBreaker`) 选择要加载的配置类。 如何加载的我们这里简单说一下。



#### @SpringFactoryImportSelector<T> 源码简单讲解

```java
public abstract class SpringFactoryImportSelector<T>
      implements DeferredImportSelector, BeanClassLoaderAware, EnvironmentAware {
	/**
	* 省略部分代码
	*/
   /**
   * 构造方法
   */
   @SuppressWarnings("unchecked")
   protected SpringFactoryImportSelector() {
       //通过spring的指定方法来获取泛型的class注解类。
       //在EnableCircuitBreakerImportSelector中指的就是 EnableCircuitBreaker
      this.annotationClass = (Class<T>) GenericTypeResolver
            .resolveTypeArgument(this.getClass(), SpringFactoryImportSelector.class);
   }

   /**
   *  选择配置类的方法
   */
   @Override
   public String[] selectImports(AnnotationMetadata metadata) {
      //首先判断是否自动装配，如果不自动装配则返回空数组
      if (!isEnabled()) {
         return new String[0];
      }
      AnnotationAttributes attributes = AnnotationAttributes.fromMap(
            metadata.getAnnotationAttributes(this.annotationClass.getName(), true));

      Assert.notNull(attributes, "No " + getSimpleName() + " attributes found. Is "
            + metadata.getClassName() + " annotated with @" + getSimpleName() + "?");

      //根据泛型中的注解类来查找所需要自动装配的类，在这里指EnableCircuitBreaker下的
      //需要自动装配的类
      List<String> factories = new ArrayList<>(new LinkedHashSet<>(SpringFactoriesLoader
            .loadFactoryNames(this.annotationClass, this.beanClassLoader)));

      if (factories.isEmpty() && !hasDefaultFactory()) {
         throw new IllegalStateException("Annotation @" + getSimpleName()
               + " found, but there are no implementations. Did you forget to include a starter?");
      }

      if (factories.size() > 1) {
         this.log.warn("More than one implementation " + "of @" + getSimpleName()
               + " (now relying on @Conditionals to pick one): " + factories);
      }

      return factories.toArray(new String[factories.size()]);
   }
```

在这里我们可以看到 通过 

```java
   List<String> factories = new ArrayList<>(new LinkedHashSet<>(SpringFactoriesLoader
            .loadFactoryNames(this.annotationClass, this.beanClassLoader)));
```

这段内容就是获取自动装配组件的方法。也就是我们spring.factories中配置的 通过固定的key去获取value。

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.hystrix.HystrixAutoConfiguration,\
org.springframework.cloud.netflix.hystrix.security.HystrixSecurityAutoConfiguration

org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker=\
org.springframework.cloud.netflix.hystrix.HystrixCircuitBreakerConfiguration
```

我们可以看到在spring-cloud-Hystrix中，这里的``org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker` 就是上面提到的那个注解。因此`HystrixCircuitBreakerConfiguration`在spring- boot启动时，就会自动装配了。



## 自动装配HystrixCircuitBreakerConfiguration

既然我们启动了`@EnableCircuitBreaker`注解，那么我们就来看看这个自动装配都装配了哪些组件。

```java
@Configuration
public class HystrixCircuitBreakerConfiguration {

    
   /**
   * 声明关于 HystrixCommand 的切面
   */
   @Bean
   public HystrixCommandAspect hystrixCommandAspect() {
      return new HystrixCommandAspect();
   }

    /**
    * 当程序关闭时，确保释放资源的bean
    */
   @Bean
   public HystrixShutdownHook hystrixShutdownHook() {
      return new HystrixShutdownHook();
   }

   @Bean
   public HasFeatures hystrixFeature() {
      return HasFeatures
            .namedFeatures(new NamedFeature("Hystrix", HystrixCommandAspect.class));
   }

   /**
    * {@link DisposableBean} that makes sure that Hystrix internal state is cleared when
    * {@link ApplicationContext} shuts down.
    */
   private class HystrixShutdownHook implements DisposableBean {

      @Override
      public void destroy() throws Exception {
         // Just call Hystrix to reset thread pool etc.
         Hystrix.reset();
      }

   }

}
```



在这源码中，我们只需要关注`HystrixCommandAspect`切面就可以了。这个就是Hystrix熔断实现机制的重中之重。

我们直接来看 `HystrixCommandAspect`切面类的实现。



```java
@Aspect
public class HystrixCommandAspect {

    //声明熔断注解对应的 工厂类map
    private static final Map<HystrixPointcutType, MetaHolderFactory> META_HOLDER_FACTORY_MAP;

    //初始化工厂类map
    static {
        META_HOLDER_FACTORY_MAP = ImmutableMap.<HystrixPointcutType, MetaHolderFactory>builder()
                .put(HystrixPointcutType.COMMAND, new CommandMetaHolderFactory())
                .put(HystrixPointcutType.COLLAPSER, new CollapserMetaHolderFactory())
                .build();
    }

 /**
 * 声明切点 
 */  @Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand)")

    public void hystrixCommandAnnotationPointcut() {
    }

    /**
 * 声明切点 
 */  @Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCollapser)")
    public void hystrixCollapserAnnotationPointcut() {
    }
    
    //省略部分代码
}
```

我们可以看到，在这段截取的代码中，我们进行了初始化工厂，以及设置切点两个操作。从这里我们可以看到，我们不光可以拦截 `@HystrixCommand`注解，也可以拦截`@HystrixCollapser`注解。



我们接着往下看另一段源码。

```java
//拦截两个切点
@Around("hystrixCommandAnnotationPointcut() || hystrixCollapserAnnotationPointcut()")
public Object methodsAnnotatedWithHystrixCommand(final ProceedingJoinPoint joinPoint) throws Throwable {
    //获取拦截的方法
    Method method = getMethodFromTarget(joinPoint);
    Validate.notNull(method, "failed to get method from joinPoint: %s", joinPoint);
    //这里注意一下，如果标注了两个注解的话，那么会抛出异常
    if (method.isAnnotationPresent(HystrixCommand.class) && method.isAnnotationPresent(HystrixCollapser.class)) {
        throw new IllegalStateException("method cannot be annotated with HystrixCommand and HystrixCollapser " +
                "annotations at the same time");
    }
    
    //通过方法标注的注解来确定MetaHolderFactory工厂对象
    MetaHolderFactory metaHolderFactory = META_HOLDER_FACTORY_MAP.get(HystrixPointcutType.of(method));
    //获取对应注解的metaHolder 这个metaHolder 是存储当前方法所需要的所有必要信息的对象
    MetaHolder metaHolder = metaHolderFactory.create(joinPoint);
    
    //通过HystrixCommandFactory获取响应的执行对象
    //这个在下面会有详细讲解
    HystrixInvokable invokable = HystrixCommandFactory.getInstance().create(metaHolder);
    
    //获取执行模式
    ExecutionType executionType = metaHolder.isCollapserAnnotationPresent() ?
            metaHolder.getCollapserExecutionType() : metaHolder.getExecutionType();

    Object result;
    try {
        //选择是否使用rxJava模式
        if (!metaHolder.isObservable()) {
            result = CommandExecutor.execute(invokable, executionType, metaHolder); 
        } else {
            result = executeObservable(invokable, executionType, metaHolder);
        }
    } catch (HystrixBadRequestException e) {
        throw e.getCause();
    } catch (HystrixRuntimeException e) {
        throw hystrixRuntimeExceptionToThrowable(metaHolder, e);
    }
    return result;
}
```

这段代码用了AspectJ的注解，来处理拦截逻辑，如果不了解AspectJ 可以自行百度查看了解一下。

这里需要重点关注的就是

```java
    HystrixInvokable invokable = HystrixCommandFactory.getInstance().create(metaHolder);
```

这段内容是获取执行熔断的执行对象，我们可以看一下`HystrixCommandFactory.getInstance().create`的实现方法，一眼就可以看出端倪。



```java
public HystrixInvokable create(MetaHolder metaHolder) {
    HystrixInvokable executable;
    //如果是 @Collapser 注解则返回 CommandCollapser
    if (metaHolder.isCollapserAnnotationPresent()) {
        executable = new CommandCollapser(metaHolder);
        //如果是 @HystrixCommand 并且是RxJAVA实现方式，则返回GenericObservableCommand
        //使用线程池来实现逻辑
    } else if (metaHolder.isObservable()) {
        executable = new GenericObservableCommand(HystrixCommandBuilderFactory.getInstance().create(metaHolder));    
        //否则返回GenericCommand
    } else {
        executable = new GenericCommand(HystrixCommandBuilderFactory.getInstance().create(metaHolder));
    }
    return executable;
}
```

在这里我们只看 `@HystrixCommand`相关的处理。 我们来看一下简单的执行对象`GenericCommand`。



```java
/**
 * Implementation of AbstractHystrixCommand which returns an Object as result.
 */
@ThreadSafe
public class GenericCommand extends AbstractHystrixCommand<Object> {

    private static final Logger LOGGER = LoggerFactory.getLogger(GenericCommand.class);

    public GenericCommand(HystrixCommandBuilder builder) {
        super(builder);
    }

    /**
     * {@inheritDoc}
     */
    @Override
    protected Object run() throws Exception {
        LOGGER.debug("execute command: {}", getCommandKey().name());
        return process(new Action() {
            @Override
            Object execute() {
                return getCommandAction().execute(getExecutionType());
            }
        });
    }

    /**
     * The fallback is performed whenever a command execution fails.
     * Also a fallback method will be invoked within separate command in the case if fallback method was annotated with
     * HystrixCommand annotation, otherwise current implementation throws RuntimeException and leaves the caller to deal with it
     * (see {@link super#getFallback()}).
     * The getFallback() is always processed synchronously.
     * Since getFallback() can throw only runtime exceptions thus any exceptions are thrown within getFallback() method
     * are wrapped in {@link FallbackInvocationException}.
     * A caller gets {@link com.netflix.hystrix.exception.HystrixRuntimeException}
     * and should call getCause to get original exception that was thrown in getFallback().
     *
     * @return result of invocation of fallback method or RuntimeException
     */
    @Override
    protected Object getFallback() {
        final CommandAction commandAction = getFallbackAction();
        if (commandAction != null) {
            try {
                return process(new Action() {
                    @Override
                    Object execute() {
                        MetaHolder metaHolder = commandAction.getMetaHolder();
                        Object[] args = createArgsForFallback(metaHolder, getExecutionException());
                        return commandAction.executeWithArgs(metaHolder.getFallbackExecutionType(), args);
                    }
                });
            } catch (Throwable e) {
                LOGGER.error(FallbackErrorMessageBuilder.create()
                        .append(commandAction, e).build());
                throw new FallbackInvocationException(unwrapCause(e));
            }
        } else {
            return super.getFallback();
        }
    }

}
```

看到这个类的方法的时候是不是特别的熟悉，像不像之前我们继承`com.netflix.hystrix.HystrixCommand`来实现客户端的熔断时所写的类。如果我们在细看一下`GenericCommand`他的父类，我们就更能理解了。

```java
@ThreadSafe
public abstract class AbstractHystrixCommand<T> extends com.netflix.hystrix.HystrixCommand<T> {
     //........
}
```

我们看到 这个父类就是继承了`com.netflix.hystrix.HystrixCommand`来实现的。



## 总结

  针对hystrix的熔断机制，简单的做了源码解析，虽然讲的不是很全很详细，但是重点的部分已经涉及到，感兴趣的同学可以继续尝试在看看。

其实Hystrix的主要实现就是基于HystrixCommandAspect的。

HystrixCommandAspect的主要职责包括

- 拦截标注`@HystrixCommand`或 `@HystrixCollapser`的方法 (@Around拦截方式)
- 生成拦截方法原信息MetaHolder
- 生成执行对象 HystrixInvokable
- 选择执行模式执行 (rxjava)