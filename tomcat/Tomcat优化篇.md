# Tomcat优化篇

  相信Web开发的小伙伴的日常开发基本离不开tomcat,Tomcat作为一个免费的开放源代码的Web 应用服务器,它的性能已经相当出色了，但是有些时候想要发挥tomcat最佳的性能还是需要一定的优化配置工作。在这我就简单的总结一下Tomcat的优化，以便以后开发工作调优等情况下的知识储备。



## 优化大纲

- 内存优化
- 线程优化
- 配置优化



### 1.内存优化

Tomcat的内存优化就是对JVM调优的一种实现。首先我们需要找到 /bin/catalina.sh 

这里我建议最好在这个命令之前来添加JVM 相关的调优参数，如图所示

位置.png

调优.png



我们可以随意调整参数来调整tomcat内存以及JVM其他配置，来保证Tomcat的优化。



### 2.线程优化

具体的线程优化可以参照tomcat文档，目录在 `\webapps\docs\config\http.html`  这里只介绍几个重要的参数。由于是tomcat8+ ，默认线程连接使用的是NIO或NIO2。

|       参数        |                             解释                             |
| :---------------: | :----------------------------------------------------------: |
| `maxConnections`  | 服务器所能接受最大的请求和处理的连接数，NIO和NIO2默认为10000，APR默认是8192. |
|   `maxThreads`    |               同一时间点上处理线程的最大数量。               |
|   `acceptCount`   | 当所有可能的请求处理线程都在使用时，传入连接请求的最大队列长度。 队列已满时收到的任何请求都将被拒绝。 默认值为100。 |
| `minSpareThreads` |             最小空闲线程数，默认为10，不易过小。             |

主要的线程优化就这些，还有其他不重要的可以自己看文档。

我们在`\conf\server.xml`中可以看到 把参数添加到如图的配置中即可。

线程优化.png



### 3.配置优化

有时候只有线程优化也是不够的。我们要了解tomcat的其他配置才可以做到完美。

#### 参数配置

|     参数     |                             解释                             |          对应文档位置          |
| :----------: | :----------------------------------------------------------: | :----------------------------: |
| `autoDeploy` | 当设置为true时，tomcat会在启动时，定时的去检查是否有新的项目更新。默认为true。因此我们需要将这个参数设置为false。 | /webapps/docs/config/host.html |



我们同样需要在`\conf\server.xml`中配置。

配置调优.png



#### APR配置

我们先打开文档，找到tomcat下的`/webapps/docs/apr.html` 。

通过介绍我们能了解到 APR( Apache Portable Runtime) 提供卓越的伸缩性，更强的性能以及跟原生服务器技术更好的集成。

接下来让我们按照文档来配置一下ARP。

我们会在linux系统下操作，如果使用的Windows可以自己按照文档操作即可。



##### 1.首先我们要安装基础依赖

我们可以看到需要的基础依赖有

Requirements.png



安装依赖

```shell
yum install apr-devel
yum install openssl-devel
yum install gcc
yum install make
```



##### 2.安装APR包

进入 apache-tomcat-8.5.39/bin 执行命令

```shell
tar -xzvf tomcat-native.tar.gz 
```

我们可以看到在bin目录下有这个文件夹

apr1.png



输入命令 `cd tomcat-native-1.2.21-src/native/` 

进入到native文件夹下。

执行命令 

```shell
./configure && make && make install
```

安装后会出现这个界面，表示安装成功，并给出了安装后的目录位置

apr2.png



通过上图我们可以看到 我们还需要配置两个环境变量 

```shell
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/apr/lib
export LD_RUN_PATH=$LD_RUN_PATH:/usr/local/apr/lib
```



输入命令使环境变量生效

```shell
source /etc/profile
```



##### 3.修改server.xml 配置

在修改之前 我们可以先来仔细看一下 server.xml。

arp3.png



我们可以看到这段注释中，已经告诉我们了如何配置 APR。我们可以仿照这个配置来修改我们`Connector`配置。

apr4.png



##### 4.启动tomcat并查看日志

进入到 /bin 目录 执行 

```shell
sh startup.sh
```



然后我们查看日志 

进入到 /logs目录 执行命令 

```shell
tail -f -n 200 catalina.out
```



apr5.png



我们看到 tomcat启动时，已经使用的APR模式。





## 小结

 对于tomcat的优化，在一般开发环境中可能不用太关注，但是实际生产环境中，由于机器的配置是固定的，我们必须想法设法通过配置各种参数来榨取机器的资源来提升程序的性能。