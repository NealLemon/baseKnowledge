# Spring Cloud系列--Ribbon源码（二）

## 前言

上一篇提到了很多核心的接口，但是没有具体走通源码的流程，现在让我们debugger一遍源码。

我们拿之前搭建的环境来做debugger环境 [Spring Cloud系列--简单实现Ribbon负载均衡](https://www.jianshu.com/p/e828b63a5cd8) 在之前讲解装配的组件上添加debugger点。

## 源码跟踪前的准备

我们知道Ribbon是客户端实现的负载均衡，而客户端又是通过`RestTemplate` 这个类来调用的，我们先来看一下这个类。

```java
public class RestTemplate extends InterceptingHttpAccessor implements RestOperations {
 //省略代码部分
}
```

我们看到了`RestTemplate`继承了 `InterceptingHttpAccessor`类，当我们使用`RestTemplate`做HTTP请求的时候，进行拦截，会执行里面的内部代码。

```java
@Nullable
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
 @Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {

 Assert.notNull(url, "URI is required");
 Assert.notNull(method, "HttpMethod is required");
 ClientHttpResponse response = null;
 try {
 ClientHttpRequest request = createRequest(url, method);
 if (requestCallback != null) {
 requestCallback.doWithRequest(request);
 }

 //请求调用，LoadBalancerInterceptor拦截器会拦截
 response = request.execute();
 handleResponse(url, method, response);
 return (responseExtractor != null ? responseExtractor.extractData(response) : null);
 }
 catch (IOException ex) {
 String resource = url.toString();
 String query = url.getRawQuery();
 resource = (query != null ? resource.substring(0, resource.indexOf('?')) : resource);
 throw new ResourceAccessException("I/O error on " + method.name() +
 " request for \"" + resource + "\": " + ex.getMessage(), ex);
 }
 finally {
 if (response != null) {
 response.close();
 }
 }
}
```

## 开始分析跟踪

既然我们知道了是哪个拦截器拦截的，那么我们就可以直接去`LoadBalancerInterceptor` 看看源码。

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

 private LoadBalancerClient loadBalancer;

 private LoadBalancerRequestFactory requestFactory;

 //省略部分代码

 @Override
 public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
 final ClientHttpRequestExecution execution) throws IOException {
 //获取URI 
 final URI originalUri = request.getURI();
 //获取 serverName
 String serviceName = originalUri.getHost();
 Assert.state(serviceName != null,
 "Request URI does not contain a valid hostname: " + originalUri);
 //拦截器RibbonLoadBalancerClient 执行
 return this.loadBalancer.execute(serviceName,
 //LoadBalancerRequestFactory 封装一层Request
 this.requestFactory.createRequest(request, body, execution));
 }

}
```

我们可以先看一下 `requestFactory.createRequest(request, body, execution)`方法。

```java
public LoadBalancerRequest<ClientHttpResponse> createRequest(
 final HttpRequest request, final byte[] body,
 final ClientHttpRequestExecution execution) {
 //这里使用lambda表达 封装了ServiceInstance 对象 
 //具体不延伸讲解 具体就是把请求的信息都放在了ServiceInstance对象里
 return instance -> {
 //封装request
 HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance,
 this.loadBalancer);
 if (this.transformers != null) {
 for (LoadBalancerRequestTransformer transformer : this.transformers) {
 serviceRequest = transformer.transformRequest(serviceRequest,
 instance);
 }
 }
 //拦截器执行链
 return execution.execute(serviceRequest, body);
 };
}
```

这个方法就是封装了一层request，将封装的结果`ServiceRequestWrapper`放入执行链内执行。默认执行链就是

`org.springframework.http.client.InterceptingClientHttpRequest.InterceptingRequestExecution#execute` 我们直接看execute方法。

```java
@Override
public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
 if (this.iterator.hasNext()) {
 ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
 return nextInterceptor.intercept(request, body, this);
 }
 else {
 HttpMethod method = request.getMethod();
 Assert.state(method != null, "No standard HTTP method");
 //创建ClientHttpRequest 
 ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
 request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
 if (body.length > 0) {
 if (delegate instanceof StreamingHttpOutputMessage) {
 StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
 streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
 }
 else {
 StreamUtils.copy(body, delegate.getBody());
 }
 }

 return delegate.execute();
 }
}
```

在这里我们可以看到

```java
ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
```

这段代码，这段代码很重要，这就是获取请求的URI。我们来进一步看这个方法的实现。

```java
public class ServiceRequestWrapper extends HttpRequestWrapper {

 private final ServiceInstance instance;

 //RibbonLoadBalancerClient
 private final LoadBalancerClient loadBalancer;

 public ServiceRequestWrapper(HttpRequest request, ServiceInstance instance,
 LoadBalancerClient loadBalancer) {
 super(request);
 this.instance = instance;
 this.loadBalancer = loadBalancer;
 }

 //获取URI
 @Override
 public URI getURI() {
 //调用RibbonLoadBalancerClient 中的 reconstructURI 方法 重新组装URI
 URI uri = this.loadBalancer.reconstructURI(this.instance, getRequest().getURI());
 return uri;
 }

}
```

##### 重头戏

在层层调用之后 我们最终还是回到了之前的 `org.springframework.cloud.client.loadbalancer.LoadBalancerInterceptor#intercept` 方法中。

执行`org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#execut()`方法。

```java
 public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
 throws IOException {
 //获取 默认负载均衡组件 ZoneAwareLoadBalancer
 ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
 //执行负载均衡
 Server server = getServer(loadBalancer, hint);
 if (server == null) {
 throw new IllegalStateException("No instances available for " + serviceId);
 }
 RibbonServer ribbonServer = new RibbonServer(serviceId, server,
 isSecure(server, serviceId),
 serverIntrospector(serviceId).getMetadata(server));

 return execute(serviceId, ribbonServer, request);
}
```

这里最重要的两点就是 获取负载均衡器 和执行负载均衡。获取负载均衡器这个是可以外部化配置的，具体源码实现这里不讲解，感兴趣的同学可以去debugger进去看一下就明白了。

我们来看一下负载均衡方法内部具体实现。
```java
protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
 if (loadBalancer == null) {
 return null;
 }
 // Use 'default' on a null hint, or just pass it on?
 return loadBalancer.chooseServer(hint != null ? hint : "default");
}
```

这里调用了 负载均衡器的chooseServer方法，也就进而进行了客户端的负载均衡。

我们这里就不讲解具体的负载均衡器的相关算法。感兴趣的同学可以看 程序猿DD大佬的书 《Spring Cloud 微服务实战》--第四章的内容。

## 总结

这里讲解的是我看程序猿DD大佬书中没有的，或者是理解不深的地方，如果看这篇博文不懂的童靴可以先看一遍《Spring Cloud 微服务实战》，如果有书中不解的地方可以回过来看看这篇文章。

