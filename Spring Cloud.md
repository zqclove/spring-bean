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
<dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
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

#是否向服务注册中心注册自己
eureka.client.register-with-eureka=false

#是否检索服务
eureka.client.fetch-registry=false

#服务注册中心的配置内容，指定服务注册中心的位置
eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka/
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

# 参考资料

[Spring Cloud Eureka详解]: https://blog.csdn.net/sunhuiliang85/article/details/76222517

