# Spring Cloud系列--Feign整合(一)

好久没更新了，由于工作杂事比较多，时间比较琐碎，所以学习记录延迟更新了。话不多说，让我们言归正传。



## 基础内容

#### [Declarative REST Client: Feign](https://cloud.spring.io/spring-cloud-openfeign/spring-cloud-openfeign.html#spring-cloud-feign) （声明式REST服务调用）

  通过spring官方文档可以了解到，Feign是一个声明式web 服务调用服务，他使得一切web服务得以简化。我们只需要创建一个接口并用注解和JAX-RS注解的方式来配置它，即可完成对服务提供方的接口绑定。



#### 远程过程调用协议（RPC）

> RPC（Remote Procedure Call）—[远程过程调用](https://baike.baidu.com/item/%E8%BF%9C%E7%A8%8B%E8%BF%87%E7%A8%8B%E8%B0%83%E7%94%A8/7854346)，它是一种通过[网络](https://baike.baidu.com/item/%E7%BD%91%E7%BB%9C/143243)从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。[RPC协议](https://baike.baidu.com/item/RPC%E5%8D%8F%E8%AE%AE)假定某些[传输协议](https://baike.baidu.com/item/%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE/8048821)的存在，如TCP或UDP，为通信程序之间携带信息数据。在OSI[网络通信](https://baike.baidu.com/item/%E7%BD%91%E7%BB%9C%E9%80%9A%E4%BF%A1/9636548)模型中，RPC跨越了[传输层](https://baike.baidu.com/item/%E4%BC%A0%E8%BE%93%E5%B1%82/4329536)和[应用层](https://baike.baidu.com/item/%E5%BA%94%E7%94%A8%E5%B1%82/4329788)。RPC使得开发包括网络[分布式](https://baike.baidu.com/item/%E5%88%86%E5%B8%83%E5%BC%8F)多程序在内的应用程序更加容易。                                                                                                         -----  百度百科

 RPC代表

- JAVA RMI (二进制协议)
- WebService (文本协议)

##### 消息传递

​	RPC是一种请求-响应协议，一次RPC在客户端初始化，再由客户端将请求消息传递到远程的服务端，执行指定的带有参数的过程。经过远程服务端执行过后，将结果作为响应内容返回到客户端。



##### 存根

在一次分布式计算RPC中，客户端和服务端转换参数的一段代码，由于存根的参数转化，RPC执行过程如同本地执行函数调用。存根必须在客户端和服务端两端均装载，并且必须保持兼容。



## 简单实现Feign整合​	

#### 1.流程图

structure.png



我们可以看到，客户端通过调用一定的Feign协议就可以实现请求和响应。接下来让我们来手动操作一下，感受一下Feign的奇妙吧。



#### 2.搭建环境并整合Feign

我们还是老规矩，基于上一套的环境基础上来开发，如果同学不会搭建之前的环境，可以看这篇文章--[Spring Cloud系列--简单实现Ribbon负载均衡](https://www.jianshu.com/p/e828b63a5cd8)，很简单就搭建起来了（如果时间充裕，可以先将[Spring Cloud系列--简单实现Hystrix熔断器](https://www.jianshu.com/p/c3ee608468e2)  实现一下）。



##### a.注册中心

​	注册中心不需要做任何改动，按照之前的文章搭建即可。



##### b.客户端（myribbon）

首先在pom.xml文件中，增加相关依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

由于是整合Feign，那么我们在路径`com.consumer.myribbon`下创建一个包名为`feign`。

（1）首先定义一个存根（上面有讲到）  ,这个存根很重要，在服务端和客户端都需要有。

```java
/**
 * @ClassName Student
 * @Description DTO
 * @Author Neal
 * @Date 2019/4/29 14:40
 * @Version 1.0
 */
public class Student {

    private String name;

    private String id;

    public Student() {
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    @Override
    public String toString() {
        return "Student{" +
                "name='" + name + '\'' +
                ", id='" + id + '\'' +
                '}';
    }
}
```



（2）定义Feign API

```java
/**
 * @ClassName StudentService
 * @Description feign API接口
 * @Author Neal
 * @Date 2019/4/29 14:36
 * @Version 1.0
 */
//服务端ID,我们的服务端，也就是provider项目的项目ID就是RIBBON-SERVICE
//下面有提及
@FeignClient(name = "RIBBON-SERVICE")  
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



（3）创建一个controller 实现对feign的调用

```java
/**
 * @ClassName StudentController
 * @Description 对feign的调用
 * @Author Neal
 * @Date 2019/4/29 14:47
 * @Version 1.0
 */
@RestController
public class StudentController implements StudentService{

    @Autowired
    private StudentService studentService;


    @Override
    public String getAllStudent() {
        return studentService.getAllStudent();
    }

    @Override
    public String saveStudent(Student student) {
        return studentService.saveStudent(student);
    }
}
```



(4) 	开启feign的支持

​	我们在启动引导类中添加`@EnableFeignClients`注解。

```java
@EnableEurekaClient
@SpringBootApplication
@EnableFeignClients(clients = StudentService.class) //指定API开启
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



最终目录如下:

目录1.png



我们客户端的修改已经完成，看看是不是很简单。



##### c.服务端（provider）

(1)  在路径`com.consumer.myribbon`下创建 包名为 `feign`的文件夹。并将存根(`Student.java`) 以及 Feign协议(`StudentService.java`)放入包下。这里需要注意一下，我们在服务端的话，就不需要标注`@FeignClient`注解了。

```java
/**
 * @ClassName StudentService
 * @Description feign API接口
 * @Author Neal
 * @Date 2019/4/29 14:36
 * @Version 1.0
 */
public interface StudentService {

    @GetMapping(value = "/myfeign/student")
    String getAllStudent();

    @PostMapping(value = "/myfeign/student")
    String saveStudent(@RequestBody Student student);
}
```



(2) 实现`StudentService`接口,这里就简单的使用内存来模拟数据库存储。

```java
/**
 * @ClassName StudentServiceImp
 * @Description StudentService实现类
 * @Author Neal
 * @Date 2019/4/29 17:10
 * @Version 1.0
 */
@Service("iStudentServiceImp")
public class StudentServiceImp implements StudentService {

    private static List<Student> studentList = new ArrayList<>();

    @Override
    public String getAllStudent() {
        return studentList.toString();
    }

    @Override
    public String saveStudent(Student student) {
        studentList.add(student);
        return "ok";
    }
}
```



(3) 服务端的调用controller

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

    @Override
    public String getAllStudent() {
        return studentService.getAllStudent();
    }

    @Override
    public String saveStudent(Student student) {
        return studentService.saveStudent(student);
    }
}
```



(4) 启动类不变

最终的文件目录如下:

目录2.png



#### 3.启动测试

首先我们启动这三个项目，启动顺序是 spring-cloud-test-register -> provider -> myribbon 

a.启动完成后，查看 http://localhost:1111/ 注册中心信息 

注册中心.png



b.如果如上图所示，则表示注册成功。接下来让我们测试Feign API接口。

###### saveStudent接口测试

打开postman 输入相应的参数并发送。

设置参数：

postman1.png

postman2.png



发送请求，并查看结果

postman3.png  

通过图中，我们可以看到请求响应的时间略长，具体原因抽时间去排查一下。



###### getAllStudent接口测试

输入url查看结果即可

结果.png



## 总结

​	基础的Feign 整合已经搞完了，是不是很简单，其实spring-cloud的强大，真的是越学越觉得恐怖，感谢巨人们的付出。之后会将熔断也整合进来，这样就是个比较完整的请求生态了。

