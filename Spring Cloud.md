# Spring Cloud Eureka

## Eureka服务治理体系

### 服务治理

服务治理是微服务架构中最为核心和基础的模块，它主要用来实现各个微服务实例的自动化注册和发现。

Spring Cloud Eureka是Spring Cloud Netflix微服务套件中的一部分，它基于Netflix Eureka做了二次封装。主要负责完成微服务架构中的服务治理功能。

Eureka分为 Eureka Server 和 Eureka Client ：

- Eureka Server主要给 Eureke Client 提供注册的服务注册中心；
  - Eureka Server采用心跳机制（默认60秒）将当前服务清单中超时（默认90秒）没有续约的服务剔除注册中心；
  - EurekaServer 在运行期间，会统计心跳失败的比例在15分钟之内是否低于85%(通常由于网络不稳定导致)。 Eureka Server会将当前的**实例注册信息（失去心跳）保护起来**， 让这些实例不会过期，尽可能**保护这些注册信息**。Eureka的思想是，同时保留“好数据”与“坏数据”总比丢掉任何数据要更好。

- Eureka Client 分为服务消费者和服务提供者：
  - 服务提供者将服务注册到注册中心并间隔一段时间向注册中心续约；
  - 服务消费者从注册中心获取服务清单，根据服务名进行调用，并且优先访问同一个Zone中的服务提供者；

注意：服务提供者可以同时是服务消费者，反之亦然。



Eureka服务治理体系如下：

![img](https://img-blog.csdn.net/20170727221824930?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VuaHVpbGlhbmc4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### 服务注册

​	在服务治理框架中，通常都会构建一个注册中心，每个服务单元向注册中心登记自己提供的服务，包括服务的主机与端口号、服务版本号、通讯协议等一些附加信息。注册中心按照服务名分类组织服务清单，同时还需要以心跳检测的方式去监测清单中的服务是否可用，若不可用需要从服务清单中剔除，以达到排除故障服务的效果。

### 服务发现

​	在服务治理框架下，服务间的调用不再通过指定具体的实例地址来实现，而是通过服务名发起请求调用实现。服务调用方通过服务名从服务注册中心的服务清单中获取服务实例的列表清单，通过指定的负载均衡策略取出一个服务实例位置来进行服务调用。



## Netflix Eureka介绍

​	Spirng Cloud Eureka使用Netflix Eureka来实现服务注册与发现。它既包含了服务端组件，也包含了客户端组件，并且服务端与客户端均采用java编写，所以Eureka主要适用于通过java实现的分布式系统，或是JVM兼容语言构建的系统。Eureka的服务端提供了较为完善的REST API，所以Eureka也支持将非java语言实现的服务纳入到Eureka服务治理体系中来，只需要其他语言平台自己实现Eureka的客户端程序。目前.Net平台的Steeltoe、Node.js的eureka-js-client等都已经实现了各自平台的Ereka客户端组件。

### Eureka服务端

​	Eureka服务端，即服务注册中心。它同其他服务注册中心一样，支持**高可用配置**。依托于强一致性提供良好的服务实例可用性，可以应对多种不同的故障场景。

   Eureka服务端支持集群模式部署，当集群中有分片发生故障的时候，Eureka会自动转入自我保护模式。它允许在分片发生故障的时候继续提供服务的发现和注册。当故障分片恢复时，集群中的其他分片会把他们的状态再次同步回来。集群中的不同服务注册中心通过**异步模式**互相复制各自的状态，这也意味着在给定的时间点每个实例关于所有服务的**状态可能存在不一致**的现象。

​	Eureka对可用性的要求高于一致性。

### Eureka客户端

​	Eureka客户端，主要处理服务的注册和发现。客户端服务通过注册和参数配置的方式，嵌入在客户端应用程序的代码中。在应用程序启动时，Eureka客户端向服务注册中心注册自身提供的服务，并周期性的发送心跳来更新它的服务租约。同时，他也能从服务端查询当前注册的服务信息并把它们缓存到本地并周期行的刷新服务状态。

​	

## 服务注册中心的使用

### 创建Eureka Server服务

创建一个Spring Boot工程，命名问Eureka-Server，并在pom文件中引入依赖：

```xml
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.3.1.RELEASE</version>
	<relativePath/> <!-- lookup parent from repository -->
</parent>

<properties>
	<java.version>1.8</java.version>
	<spring-cloud.version>Hoxton.SR6</spring-cloud.version>
</properties>

<dependencies>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
	</dependency>
</dependencies>

<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>${spring-cloud.version}</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```

### Eureka Server相关配置

在默认配置下，Eureka Server会将自己也作为客户端来尝试注册自己，我们需要禁用它的客户端禁用行为。

下面是一个Eureka Server的application.properites的相关配置：

```properties
#服务注册中心端口号
server.port=1110

#服务注册中心实例的主机名
eureka.instance.hostname=localhost

#是否想服务注册中心注册自身服务
eureka.client.register-with-eureka=false

#是否检索服务
eureka.client.fetch-registry=false

#服务注册中心的配置内容，指定服务注册中心的位置
eureka.client.serverUrl.defaultZone=http://${eureka.instace.hostname}:${server.port}/eureka/
```

### 启动服务中心

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}

}
```

通过main方法运行spring boot 应用。

### 高可用服务注册中心

#### 概念

​	考虑到发生故障的情况，服务注册中心发生故障必将会造成整个系统的瘫痪，因此需要保证服务注册中心的高可用。

​	Eureka Server在设计的时候就考虑了高可用设计，在Eureka服务治理设计中，所有节点既是服务的提供方，也是服务的消费方，服务注册中心也不例外。

​	Eureka Server的高可用实际上就是将自己作为服务向其他服务注册中心注册自己，这样就可以形成一组互相注册的服务注册中心，以实现服务清单的互相同步，达到高可用的效果。

#### 构建服务注册中心集群

```properties
#application-peer1.properties
server.port=1111

eureka.instance.hostname=master

#疑问，这里是设置为true还是false？
eureka.client.register-with-eureka=false 

eureka.client.fetch-registry=false

eureka.instance.preferIpAddress=true

eureka.server.enableSelfPreservation=false

eureka.client.serviceUrl.defaultZone=http://backup1:1112/eureka/,http://backup2:1113/eureka/
```

```properties
#application-peer2.properties
server.port=1112

eureka.instance.hostname=backup1

eureka.client.register-with-eureka=false

eureka.client.fetch-registry=false

eureka.instance.preferIpAddress=true

eureka.server.enableSelfPreservation=false

eureka.client.serviceUrl.defaultZone=http://master:1111/eureka/,http://backup2:1113/eureka/
```

```properties
#application-peer3.properties
server.port=1113

eureka.instance.hostname=backup2

eureka.client.register-with-eureka=false

eureka.client.fetch-registry=false

eureka.instance.preferIpAddress=true

eureka.server.enableSelfPreservation=false

eureka.client.serviceUrl.defaultZone=http://master:1111/eureka/,http://backup1:1112/eureka/
```

在hosts文件中增加如下配置

```
127.0.0.1 master

127.0.0.1 backup1

127.0.0.1 backup2
```

### 失效剔除

​	服务实例可能由于内存溢出、网络故障等原因使服务不能正常运作。而服务注册中心并未收到“服务下线”的请求，为了从服务列表将这些无法提供服务的实例剔除，Eureka Server在启动时会创建一个定时任务，默认每个一段时间（默认60s）将当前服务列表中超时（默认90s）没有续约的服务剔除出去。

### 自我保护

​	服务注册到Eureka Server之后，会维持一个心跳连接，告知Eureka Server正在提供服务。Eureka Server在运行期间会统计心跳失败的比例在15分钟之内是否低于85%，如果出现低于的情况，Eureka Server会将当前实例注册信息保护起来，让这些实例不会过期。这样做会使客户端很容易拿到实际已经不存在的服务实例，会出现调用失败的情况。因此客户端要有容错机制，比如请求重试、断路器。

以下是自我保护相关的属性：

eureka.server.enableSelfPreservation=true. 可以设置改参数值为false，以确保注册中心将不可用的实例删除


### region（地域）与zone（可用区）

​	region和zone（或者Availability Zone）均是AWS的概念。在非AWS环境下，我们可以简单地将region理解为地域，zone理解成机房。一个region可以包含多个zone，可以理解为一个地域内的多个不同的机房。不同地域的距离很远，一个地域的不同zone间距离往往较近，也可能在同一个机房内。

​	region可以通过配置文件进行配置，如果不配置，会默认使用us-east-1。同样Zone也可以配置，如果不配置，会默认使用defaultZone。

Eureka Server通过eureka.client.serviceUrl.defaultZone属性设置Eureka的服务注册中心的位置。

指定region和zone的属性如下：

（1）eureka.client.availabilityZones.myregion=myzone # myregion是region

（2）eureka.client.region=myregion

 	Ribbon的默认策略会优先访问通客户端处于同一个region中的服务端实例，只有当同一个zone中没有可用服务端实例的时候才会访问其他zone中的实例。所以通过zone属性的定义，配合实际部署的物理结构，我们就可以设计出应对区域性故障的容错集群。

### 安全验证

我们启动了Eureka Server，然后在浏览器中输入http://localhost:8761/后，直接回车，就进入了spring cloud的服务治理页面，这么做在生产环境是极不安全的，下面，我们就给Eureka Server加上安全的用户认证.

（1）pom文件中引入依赖

```xml
<dependency> 

   <groupId>org.springframework.boot</groupId> 

   <artifactId>spring-boot-starter-security</artifactId> 

</dependency> 
```

（2）serviceurl中加入安全校验信息

```properties
eureka.client.serviceUrl.defaultZone=http://<username>:<password>@${eureka.instance.hostname}:${server.port}/eureka/
```

### Eureka信息面板

服务启动后，访问http://127.0.0.1:1110/可以看到Eureka的信息面板。如下图,目前Instancescurrently registered with Eureka一栏显示注册到服务注册中心内的服务。

![img](https://img-blog.csdn.net/20170727222118909?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VuaHVpbGlhbmc4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)



## 服务提供者

### 服务注册

​	服务提供者在启动的时候会通过REST请求的方式将自己注册到Eureka Server上，同时带上自身服务的一些元数据信息。Eureka Server接收到这个REST请求之后，将元数据信息存储在一个双层结构的Map中，其中第一层的key是服务名。第二层的key 是具体服务的实例名。

​	在服务注册时，需要确认一下eureka.client.register-with-eureka=true参数是否正确，该值默认为true。若设置为fasle将不会启动注册操作。

### 服务同步

​	从eureka服务治理体系架构图中可以看到，不同的服务提供者可以注册在不同的服务注册中心上，它们的信息被不同的服务注册中心维护。

​	此时，由于多个服务注册中心互相注册为服务，当服务提供者发送注册请求到一个服务注册中心时，它会将该请求转发给集群中相连的其他注册中心，从而实现服务注册中心之间的服务同步。通过服务同步，提供者的服务信息就可以通过集群中的任意一个服务注册中心获得。

### 服务续约

​	在注册服务之后，服务提供者会维护一个心跳用来持续高速Eureka Server，“我还在持续提供服务”，否则Eureka Server的剔除任务会将该服务实例从服务列表中排除出去。我们称之为服务续约。

下面是服务续约的两个重要属性：

（1）eureka.instance.lease-expiration-duration-in-seconds

**leaseExpirationDurationInSeconds**，表示eureka server至上一次收到client的心跳之后，等待下一次心跳的超时时间，在这个时间内若没收到下一次心跳，则将移除该instance。

- 默认为90秒
- 如果该值太大，则很可能将流量转发过去的时候，该instance已经不存活了。
- 如果该值设置太小了，则instance则很可能因为临时的网络抖动而被摘除掉。
- 该值至少应该大于leaseRenewalIntervalInSeconds

（2）eureka.instance.lease-renewal-interval-in-seconds

**leaseRenewalIntervalInSeconds**，表示eureka client发送心跳给server端的频率。如果在leaseExpirationDurationInSeconds后，server端没有收到client的心跳，则将摘除该instance。除此之外，如果该instance实现了HealthCheckCallback，并决定让自己unavailable的话，则该instance也不会接收到流量。

- 默认30秒

### 创建并注册服务提供者

#### 创建Eureka Client服务

创建一个Spring Boot工程，命名问Eureka-Client，并在pom文件中引入依赖：

```xml
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.3.1.RELEASE</version>
	<relativePath /> <!-- lookup parent from repository -->
</parent>

<properties>
	<java.version>1.8</java.version>
	<spring-cloud.version>Hoxton.SR6</spring-cloud.version>
</properties>

<dependencies>
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
	</dependency>

	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
</dependencies>

<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>${spring-cloud.version}</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>	</dependencyManagement>
```

#### Eureka Client相关配置

在Eureka客户端中需要在application.properties文件中指定服务注册中心地址

```properties
#指定服务端口
server.port=1120

#指定服务名称
spring.application.name=hello-service

#指定注册中心端口
eureka.client.proxy-port=1110

#指定注册中心实例名称
eureka.instance.hostname=localhost

#指定服务注册中心地址
eureka.client.service-url.defaultZone=http://${eureka.instance.hostname}:${eureka.client.proxy-port}/eureka/
```

#### 启动Eureka客户端

```java
@SpringBootApplication
@EnableEurekaClient
public class EurekaClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaClientApplication.class, args);
	}

}
```

#### 遇到的问题

```
问题描述：在启动Eureka客户端后立即下线服务了

日志打印错误：Unregistering application xxx with eureka with status DOWN

解决方案：导入spring-boot-starter-web依赖即可解决
```

## 服务消费者

### 获取服务

​	消费者服务启动时，会发送一个REST请求到服务注册中心，用于获取注册中心的服务清单。为了性能考虑，Eureka Server会维护一份只读的服务注册清单来返回给服务消费者客户端，同时该缓存清单默认会间隔30秒更新一次。

下面是获取服务的两个重要的属性：

```properties
#是否需要去检索寻找服务，默认是true
eureka.client.fetch-registry
 
#表示eureka client间隔多久去拉取服务注册信息，默认为30秒，对于api-gateway，如果要迅速获取服务注册状态，可以缩小该值，比如5秒
eureka.client.registry-fetch-interval-seconds
```



### 服务调用

​	服务消费者在获取服务清单后，通过服务名可以获取具体提供服务的实例名和该实例的元数据信息。因为有这些服务实例的详细信息，所以客户端可以根据自己的需要决定具体调用哪个实例，在Ribbon中会默认采用轮询的方式进行调用，从而实现客户端的负载均衡。

### 服务下线

​	在系统运行过程中必然会面临关闭或重启服务的某个实例的情况，在服务关闭操作时，会触发一个服务下线的REST服务请求给Eureka  Server，告诉服务注册中心：“我要下线了。”服务端在接收到该请求后，将该服务状态置位下线（DOWN），并把该下线事件传播出去。

​	服务下线主要发生在服务提供者，但服务消费者也可以同时是服务提供者。

## 配置详解

 ![img](https://img-blog.csdn.net/20170727222209291?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VuaHVpbGlhbmc4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### **Eureka客户端配置**

1、RegistryFetchIntervalSeconds

   从eureka服务器注册表中获取注册信息的时间间隔（s），默认为30秒

2、InstanceInfoReplicationIntervalSeconds

   复制实例变化信息到eureka服务器所需要的时间间隔（s），默认为30秒

3、InitialInstanceInfoReplicationIntervalSeconds

   最初复制实例信息到eureka服务器所需的时间（s），默认为40秒

4、EurekaServiceUrlPollIntervalSeconds

   询问Eureka服务url信息变化的时间间隔（s），默认为300秒

5、ProxyHost

   获取eureka服务的代理主机，默认为null

6、ProxyProxyPort

   获取eureka服务的代理端口, 默认为null 

7、ProxyUserName

   获取eureka服务的代理用户名，默认为null

8、ProxyPassword

   获取eureka服务的代理密码，默认为null 

9、GZipContent

​    eureka注册表的内容是否被压缩，默认为true，并且是在最好的网络流量下被压缩

10、EurekaServerReadTimeoutSeconds

   eureka需要超时读取之前需要等待的时间，默认为8秒

11、EurekaServerConnectTimeoutSeconds

   eureka需要超时连接之前需要等待的时间，默认为5秒

12、BackupRegistryImpl

   获取实现了eureka客户端在第一次启动时读取注册表的信息作为回退选项的实现名称

13、EurekaServerTotalConnections

​    eureka客户端允许所有eureka服务器连接的总数目，默认是200

14、EurekaServerTotalConnectionsPerHost

​    eureka客户端允许eureka服务器主机连接的总数目，默认是50

15、EurekaServerURLContext

​    表示eureka注册中心的路径，如果配置为eureka，则为http://x.x.x.x:x/eureka/，在eureka的配置文件中加入此配置表示eureka作为客户端向注册中心注册，从而构成eureka集群。此配置只有在eureka服务器ip地址列表是在DNS中才会用到，默认为null

16、EurekaServerPort

​    获取eureka服务器的端口，此配置只有在eureka服务器ip地址列表是在DNS中才会用到。默认为null

17、EurekaServerDNSName

​    获取要查询的DNS名称来获得eureka服务器，此配置只有在eureka服务器ip地址列表是在DNS中才会用到。默认为null

18、UseDnsForFetchingServiceUrls

​    eureka客户端是否应该使用DNS机制来获取eureka服务器的地址列表，默认为false

19、RegisterWithEureka

​    实例是否在eureka服务器上注册自己的信息以供其他服务发现，默认为true

20、PreferSameZoneEureka

​    实例是否使用同一zone里的eureka服务器，默认为true，理想状态下，eureka客户端与服务端是在同一zone下

21、AllowRedirects

​    服务器是否能够重定向客户端请求到备份服务器。 如果设置为false，服务器将直接处理请求，如果设置为true，它可能发送HTTP重定向到客户端。默认为false

22、LogDeltaDiff

​    是否记录eureka服务器和客户端之间在注册表的信息方面的差异，默认为false

23、DisableDelta(*)

​    默认为false

24、fetchRegistryForRemoteRegions

​    eureka服务注册表信息里的以逗号隔开的地区名单，如果不这样返回这些地区名单，则客户端启动将会出错。默认为null

25、Region

​    获取实例所在的地区。默认为us-east-1

26、AvailabilityZones

​    获取实例所在的地区下可用性的区域列表，用逗号隔开。

27、EurekaServerServiceUrls

​    Eureka服务器的连接，默认为http：//XXXX：X/eureka/,但是如果采用DNS方式获取服务地址，则不需要配置此设置。

28、FilterOnlyUpInstances（*）

​    是否获得处于开启状态的实例的应用程序过滤之后的应用程序。默认为true

29、EurekaConnectionIdleTimeoutSeconds

​    Eureka服务的http请求关闭之前其响应的时间，默认为30 秒

30、FetchRegistry

​    此客户端是否获取eureka服务器注册表上的注册信息，默认为true

31、RegistryRefreshSingleVipAddress

​    此客户端只对一个单一的VIP注册表的信息感兴趣。默认为null

32、HeartbeatExecutorThreadPoolSize(*)

​    心跳执行程序线程池的大小,默认为5

33、HeartbeatExecutorExponentialBackOffBound(*)

​    心跳执行程序回退相关的属性，是重试延迟的最大倍数值，默认为10

34、CacheRefreshExecutorThreadPoolSize(*)

​    执行程序缓存刷新线程池的大小，默认为5

35、CacheRefreshExecutorExponentialBackOffBound

​    执行程序指数回退刷新的相关属性，是重试延迟的最大倍数值，默认为10

36、DollarReplacement

​    eureka服务器序列化/反序列化的信息中获取“$”符号的的替换字符串。默认为“_-”

37、EscapeCharReplacement

​    eureka服务器序列化/反序列化的信息中获取“_”符号的的替换字符串。默认为“__”

38、OnDemandUpdateStatusChange（*）

​    如果设置为true,客户端的状态更新将会点播更新到远程服务器上，默认为true

39、EncoderName

​    这是一个短暂的编码器的配置，如果最新的编码器是稳定的，则可以去除，默认为null

40、DecoderName

​    这是一个短暂的解码器的配置，如果最新的解码器是稳定的，则可以去除，默认为null

41、ClientDataAccept（*）

​    客户端数据接收

42、Experimental（*）

​    当尝试新功能迁移过程时，为了避免配置API污染，相应的配置即可投入实验配置部分，默认为null

###  **Eureka服务端配置**

   1、AWSAccessId

​    获取aws访问的id，主要用于弹性ip绑定，此配置是用于aws上的，默认为null

​    2、AWSSecretKey

​    获取aws私有秘钥，主要用于弹性ip绑定，此配置是用于aws上的，默认为null

​    3、EIPBindRebindRetries

​    获取服务器尝试绑定到候选的EIP的次数，默认为3

​    4、EIPBindingRetryIntervalMsWhenUnbound(*)

​    服务器检查ip绑定的时间间隔，单位为毫秒，默认为1 * 60 * 1000

​    5、EIPBindingRetryIntervalMs

​    与上面的是同一作用，仅仅是稳定状态检查，默认为5 * 60 * 1000

​    6、EnableSelfPreservation

​    自我保护模式，当出现出现网络分区、eureka在短时间内丢失过多客户端时，会进入自我保护模式，即一个服务长时间没有发送心跳，eureka也不会将其删除，默认为true

​    7、RenewalPercentThreshold(*)

​    ![img](https://images2015.cnblogs.com/blog/952935/201706/952935-20170623150643570-1733575926.png)

​    阈值因子，默认是0.85，如果阈值比最小值大，则自我保护模式开启

​    8、RenewalThresholdUpdateIntervalMs

​    阈值更新的时间间隔，单位为毫秒，默认为15 * 60 * 1000

​    9、PeerEurekaNodesUpdateIntervalMs(*)

​    集群里eureka节点的变化信息更新的时间间隔，单位为毫秒，默认为10 * 60 * 1000

​    10、EnableReplicatedRequestCompression

​    复制的数据在发送请求时是否被压缩，默认为false

​    11、NumberOfReplicationRetries

​    获取集群里服务器尝试复制数据的次数，默认为5

​    12、PeerEurekaStatusRefreshTimeIntervalMs

​    服务器节点的状态信息被更新的时间间隔，单位为毫秒，默认为30 * 1000

​    13、WaitTimeInMsWhenSyncEmpty(*)

​    在Eureka服务器获取不到集群里对等服务器上的实例时，需要等待的时间，单位为毫秒，默认为1000 * 60 * 5

​    14、PeerNodeConnectTimeoutMs

​    连接对等节点服务器复制的超时的时间，单位为毫秒，默认为200

​    15、PeerNodeReadTimeoutMs

​    读取对等节点服务器复制的超时的时间，单位为毫秒，默认为200

​    16、PeerNodeTotalConnections

​    获取对等节点上http连接的总数，默认为1000

​    17、PeerNodeTotalConnectionsPerHost(*)

​    获取特定的对等节点上http连接的总数，默认为500

​    18、PeerNodeConnectionIdleTimeoutSeconds(*)

​    http连接被清理之后服务器的空闲时间，默认为30秒

​    19、RetentionTimeInMSInDeltaQueue(*)

​    客户端保持增量信息缓存的时间，从而保证不会丢失这些信息，单位为毫秒，默认为3 * 60 * 1000

​    20、DeltaRetentionTimerIntervalInMs

​    清理任务程序被唤醒的时间间隔，清理过期的增量信息，单位为毫秒，默认为30 * 1000

​    21、EvictionIntervalTimerInMs

​    过期实例应该启动并运行的时间间隔，单位为毫秒，默认为60 * 1000

​    22、ASGQueryTimeoutMs（*）

​    查询AWS上ASG（自动缩放组）信息的超时值，单位为毫秒，默认为300

​    23、ASGUpdateIntervalMs

​    从AWS上更新ASG信息的时间间隔，单位为毫秒，默认为5 * 60 * 1000

​    24、ASGCacheExpiryTimeoutMs(*)

​    缓存ASG信息的到期时间，单位为毫秒，默认为10 * 60 * 1000

​    25、ResponseCacheAutoExpirationInSeconds（*）

​    当注册表信息被改变时，则其被保存在缓存中不失效的时间，默认为180秒

​    26、ResponseCacheUpdateIntervalMs（*）

​    客户端的有效负载缓存应该更新的时间间隔，默认为30 * 1000毫秒

​    27、UseReadOnlyResponseCache（*）

​    目前采用的是二级缓存策略，一个是读写高速缓存过期策略，另一个没有过期只有只读缓存，默认为true，表示只读缓存

​    28、DisableDelta（*）

​    增量信息是否可以提供给客户端看，默认为false

​    29、MaxIdleThreadInMinutesAgeForStatusReplication（*）

​    状态复制线程可以保持存活的空闲时间，默认为10分钟

​    30、MinThreadsForStatusReplication

​    被用于状态复制的线程的最小数目，默认为1

​    31、MaxThreadsForStatusReplication

​    被用于状态复制的线程的最大数目，默认为1

​    32、MaxElementsInStatusReplicationPool

​    可允许的状态复制池备份复制事件的最大数量，默认为10000

​    33、SyncWhenTimestampDiffers

​    当时间变化实例是否跟着同步，默认为true

​    34、RegistrySyncRetries

​    当eureka服务器启动时尝试去获取集群里其他服务器上的注册信息的次数，默认为5

​    35、RegistrySyncRetryWaitMs

​    当eureka服务器启动时获取其他服务器的注册信息失败时，会再次尝试获取，期间需要等待的时间，默认为30 * 1000毫秒

​    36、MaxElementsInPeerReplicationPool（*）

​    复制池备份复制事件的最大数量，默认为10000

​    37、MaxIdleThreadAgeInMinutesForPeerReplication（*）

​    复制线程可以保持存活的空闲时间，默认为15分钟

​    38、MinThreadsForPeerReplication（*）

​    获取将被用于复制线程的最小数目，默认为5

​    39、MaxThreadsForPeerReplication

​    获取将被用于复制线程的最大数目，默认为20

​    40、MaxTimeForReplication（*）

​    尝试在丢弃复制事件之前进行复制的时间，默认为30000毫秒

​    41、PrimeAwsReplicaConnections（*）

​    对集群中服务器节点的连接是否应该准备，默认为true

​    42、DisableDeltaForRemoteRegions（*）

​    增量信息是否可以提供给客户端或一些远程地区，默认为false

​    43、RemoteRegionConnectTimeoutMs（*）

​    连接到对等远程地eureka节点的超时时间，默认为1000毫秒

​    44、RemoteRegionReadTimeoutMs（*）

​    获取从远程地区eureka节点读取信息的超时时间，默认为1000毫秒

​    45、RemoteRegionTotalConnections

​    获取远程地区对等节点上http连接的总数，默认为1000

​    46、RemoteRegionTotalConnectionsPerHost

​    获取远程地区特定的对等节点上http连接的总数，默认为500

​    47、RemoteRegionConnectionIdleTimeoutSeconds

​    http连接被清理之后远程地区服务器的空闲时间，默认为30秒

​    48、GZipContentFromRemoteRegion（*）

​    eureka服务器中获取的内容是否在远程地区被压缩，默认为true

​    49、RemoteRegionUrlsWithName

​    针对远程地区发现的网址域名的map

​    50、RemoteRegionUrls

​    远程地区的URL列表

​    51、RemoteRegionAppWhitelist（*）

​    必须通过远程区域中检索的应用程序的列表

​    52、RemoteRegionRegistryFetchInterval

​    从远程区域取出该注册表的信息的时间间隔，默认为30秒

​    53、RemoteRegionFetchThreadPoolSize

​    用于执行远程区域注册表请求的线程池的大小，默认为20

​    54、RemoteRegionTrustStore

​    用来合格请求远程区域注册表的信任存储文件，默认为空

​    55、RemoteRegionTrustStorePassword

​    获取偏远地区信任存储文件的密码，默认为“changeit”

​    56、disableTransparentFallbackToOtherRegion(*)

​    如果在远程区域本地没有实例运行，对于应用程序回退的旧行为是否被禁用， 默认为false

​    57、BatchReplication(*)

​    表示集群节点之间的复制是否为了网络效率而进行批处理，默认为false

​    58、LogIdentityHeaders(*)

​    Eureka服务器是否应该登录clientAuthHeaders，默认为true

​    59、RateLimiterEnabled

​    限流是否应启用或禁用，默认为false

​    60、RateLimiterThrottleStandardClients

​    是否对标准客户端进行限流，默认false

​    61、RateLimiterPrivilegedClients（*）

​    认证的客户端列表，这里是除了标准的eureka Java客户端。

​    62、RateLimiterBurstSize（*）

​    速率限制的burst size ，默认为10，这里用的是令牌桶算法

​    63、RateLimiterRegistryFetchAverageRate(*)

​    速率限制器用的是令牌桶算法，此配置指定平均执行注册请求速率，默认为500

​    64、RateLimiterFullFetchAverageRate（*）

​    速率限制器用的是令牌桶算法，此配置指定平均执行请求速率，默认为100

​    65、ListAutoScalingGroupsRoleName（*）

​    用来描述从AWS第三账户的自动缩放组中的角色名称，默认为“ListAutoScalingGroups”

​    66、JsonCodecName（*）

​    如果没有设置默认的编解码器将使用全JSON编解码器，获取的是编码器的类名称

​    67、XmlCodecName(*)

​    如果没有设置默认的编解码器将使用xml编解码器，获取的是编码器的类名称

​    68、BindingStrategy(*)

​    获取配置绑定EIP或Route53的策略。

​    69、Route53DomainTTL（*）

​    用于建立route53域的ttl，默认为301

​    70、Route53BindRebindRetries（*）

​    服务器尝试绑定到候选Route53域的次数，默认为3

​    71、Route53BindingRetryIntervalMs（*）

​    服务器应该检查是否和Route53域绑定的时间间隔，默认为5 * 60 * 1000毫秒

​    72、Experimental(*)

​    当尝试新功能迁移过程时，为了避免配置API污染，相应的配置即可投入实验配置部分，默认为null

### **实例微服务端配置**

​    1、InstanceId

​    此实例注册到eureka服务端的唯一的实例ID,其组成为${spring.application.name}:${spring.application.instance_id:${random.value}}

​    2、Appname

​    获得在eureka服务上注册的应用程序的名字，默认为unknow

​    3、AppGroupName

​    获得在eureka服务上注册的应用程序组的名字，默认为unknow

​    4、InstanceEnabledOnit（*）

​    实例注册到eureka服务器时，是否开启通讯，默认为false

​    5、NonSecurePort

​    获取该实例应该接收通信的非安全端口。默认为80

​    6、SecurePort

​    获取该实例应该接收通信的安全端口，默认为443

​    7、NonSecurePortEnabled

​    该实例应该接收通信的非安全端口是否启用，默认为true

​    8、SecurePortEnabled

​    该实例应该接收通信的安全端口是否启用，默认为false

​    9、LeaseRenewalIntervalInSeconds

​    eureka客户需要多长时间发送心跳给eureka服务器，表明它仍然活着,默认为30 秒

​    10、LeaseExpirationDurationInSeconds

​    Eureka服务器在接收到实例的最后一次发出的心跳后，需要等待多久才可以将此实例删除，默认为90秒

​    11、VirtualHostName

​    此实例定义的虚拟主机名，其他实例将通过使用虚拟主机名找到该实例。

​    12、SecureVirtualHostName

​    此实例定义的安全虚拟主机名

​    13、ASGName（*）

​    与此实例相关联 AWS自动缩放组名称。此项配置是在AWS环境专门使用的实例启动，它已被用于流量停用后自动把一个实例退出服务。

​    14、HostName

​    与此实例相关联的主机名，是其他实例可以用来进行请求的准确名称

​    15、MetadataMap(*)

​    获取与此实例相关联的元数据(key,value)。这个信息被发送到eureka服务器，其他实例可以使用。

​    16、DataCenterInfo（*）

​    该实例被部署在数据中心

​    17、IpAddress

​    获取实例的ip地址

​    18、StatusPageUrlPath（*）

​    获取此实例状态页的URL路径，然后构造出主机名，安全端口等，默认为/info

​    19、StatusPageUrl(*)

​    获取此实例绝对状态页的URL路径，为其他服务提供信息时来找到这个实例的状态的路径，默认为null

​    20、HomePageUrlPath（*）

​    获取此实例的相关主页URL路径，然后构造出主机名，安全端口等，默认为/

​    21、HomePageUrl(*)

​    获取此实例的绝对主页URL路径，为其他服务提供信息时使用的路径,默认为null

​    22、HealthCheckUrlPath

​    获取此实例的相对健康检查URL路径，默认为/health

​    23、HealthCheckUrl

​    获取此实例的绝对健康检查URL路径,默认为null

​    24、SecureHealthCheckUrl

​    获取此实例的绝对安全健康检查网页的URL路径，默认为null

​    25、DefaultAddressResolutionOrder

​    获取实例的网络地址，默认为[]

​    26、Namespace

​    获取用于查找属性的命名空间，默认为eureka

## 服务实例类配置

### 端点配置

​	eureka实例的状态页面和健康监控的url默认为spring boot actuator提供的/info端点和/health端点。我们必须确保Eureka客户端的/health端点在发送元数据的时候，是一个能够被注册中心访问到的地址，否则服务注册中心不会根据应用的健康检查来更改状态（仅当开启了healthcheck功能时，以该端点信息作为健康检查标准）。而如果/info端点不正确的话，会导致在Eureka面板中单击服务时，无法访问到服务实例提供的信息接口。

​	大多数情况下，我们不需要修改这个几个url配置。但是当应用不使用默认的上下文(context path或servlet path，比如配置server.servletPath=/test），或者管理终端路径（比如配置management.contextPath=/admin）时，我们需要修改健康检查和状态页的url地址信息。

application.yml配置文件如下：

```properties
server.context-path=/helloeureka

#下面配置为相对路径，也支持配置成绝对路径，例如需要支持https

eureka.instance.health-check-url-path=${server.context-path}/health

eureka.instance.status-page-url-path=${server.context-path}/info
```

### 元数据

​	默认情况下，Eureka中各个服务实例的健康检测并不是通过spring-boot-acturator模块的/health端点来实现的，而是依靠客户端心跳的方式来保持服务实例的存活。在Eureka的服务续约与剔除机制下，客户端的健康状态从注册到注册中心开始都会处于UP状态，除非心跳终止一段时间之后，服务注册中心将其剔除。默认的心跳实现方式可以有效检查客户端进程是否正常运作，但却无法保证客户端应用能够正常提供服务。

​	在Spring Cloud Eureka中，可以把Eureka客户端的健康检测交给spring-boot-actuator模块的health端点，以实现更加全面的健康状态维护，设置方式如下：

（1）      在pom.xml中引入spring-boot-starter-actuator模块的依赖

（2）      在application.properties中增加参数配置eureka.client.healthcheck.enabled=true

### 其他配置

除了上述配置参数外，下面整理了一些EurekaInstanceConfigBean中定义的配置参数以及对应的说明和默认值，这些参数均以eureka.instance为前缀。

![img](https://img-blog.csdn.net/20170727222240599?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VuaHVpbGlhbmc4NQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

## 通讯协议

​	默认情况下，Eureka使用Jersey和XStream配合JSON作为Server与Client之间的通讯协议。也可以选择实现自己的协议来代替。 

## 与Zookeeper对比

### 1、分布式系统的CAP理论：

一致性（C）：所有的节点上的数据时刻保持同步。

可用性（A）：每个请求都能接受到一个响应，无论响应成功或失败。

分区容错性（P）：系统应该能持续提供服务，即使系统内部有消息丢失（分区）。

由于分区容错性在是分布式系统中必须要保证的，因此我们只能在A和C之间进行权衡。

在此Zookeeper保证的是CP, 而Eureka则是AP。

### 2、Zookeeper保证CP

​	ZooKeeper是个  CP的，即任何时刻对ZooKeeper的访问请求能得到一致的数据结果，同时系统对网络分割具备容错性、但是它不能保证每次服务请求的可用性(注：也就是在极端环境下，ZooKeeper可能会丢弃一些请求，消费者程序需要重新请求才能获得结果)。

​	例如：当master节点因为网络故障与其他节点失去联系时，剩余节点会重新进行leader选举。问题在于，选举leader的时间太长，30 ~ 120s, 且选举期间整个zk集群都是不可用的，这就导致在选举期间注册服务瘫痪。

### 3、Eureka保证AP

​	Eureka看明白了这一点，因此在设计时就优先保证可用性。我们可以容忍注册中心返回的是几分钟以前的注册信息，但不能接受服务直接down掉不可用。也就是说，服务注册功能对可用性的要求要高于一致性。

​	如果Eureka服务节点在短时间里丢失了大量的心跳连接(注：可能发生了网络故障)，那么这个  Eureka节点会进入“自我保护模式”，同时保留那些“心跳死亡”的服务注册信息不过期。此时，这个Eureka节点对于新的服务还能提供注册服务，对于“死亡”的仍然保留，以防还有客户端向其发起请求。当网络故障恢复后，这个Eureka节点会退出“自我保护模式”。Eureka的哲学是，同时保留“好数据”与“坏数据”总比丢掉任何数据要更好。

# Spring Cloud Ribbon

### 介绍

​	Spring Cloud 提供了让服务调用端具备负载均衡能力的Ribbon，通过和Eureka的紧密结合，不用在服务集群内再架设负载均衡服务，很大程度简化了服务集群内的架构。

​	服务调用端通过获取服务清单，获取到服务提供者的详细信息，使得在服务调用端实现负载均衡。

### 配置

![img](https://images2017.cnblogs.com/blog/166781/201802/166781-20180214012425859-1316261402.png)

**详解：**

#### 1.RibbonAutoConfiguration配置生成RibbonLoadBalancerClient实例

代码位置：

jar包 ：spring-cloud-netflix-core-1.3.5.RELEASE.jar
全限定符：org.springframework.cloud.netflix.ribbon
**RibbonAutoConfiguration.class：**

```java
@Configuration
@ConditionalOnClass({ IClient.class, RestTemplate.class, AsyncRestTemplate.class, Ribbon.class})
@RibbonClients
@AutoConfigureAfter(name = "org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration")
@AutoConfigureBefore({LoadBalancerAutoConfiguration.class, AsyncLoadBalancerAutoConfiguration.class})
@EnableConfigurationProperties(RibbonEagerLoadProperties.class)
public class RibbonAutoConfiguration {

    // 略

    @Bean
    @ConditionalOnMissingBean(LoadBalancerClient.class)
    public LoadBalancerClient loadBalancerClient() {
        return new RibbonLoadBalancerClient(springClientFactory());
    }

        // 略
}
```

先看配置条件项，RibbonAutoConfiguration配置必须在LoadBalancerAutoConfiguration配置前执行，因为在LoadBalancerAutoConfiguration配置中会使用RibbonLoadBalancerClient实例。

RibbonLoadBalancerClient继承自LoadBalancerClient接口，是负载均衡客户端，也是负载均衡策略的调用方。

# 参考资料

[Spring Cloud Eureka详解]: https://blog.csdn.net/sunhuiliang85/article/details/76222517

[微服务架构：Eureka参数配置项详解]: https://www.cnblogs.com/fangfuhai/p/7070325.html
[撸一撸Spring Cloud Ribbon的原理]: https://www.cnblogs.com/kongxianghai/p/8445030.html

