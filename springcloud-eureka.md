# Spring Cloud Netflix Eureka
## 1、前微服务时代
<br>前微服务时代分布式系统基本组成
<br><br>1. 服务提供方( Provider)
<br><br>2. 服务消费方( Consumer)
<br><br>3. 服务注册中心( Registry) 
<br><br>4. 服务路由( Router)
<br><br>5. 服务代理 ( Broker )
<br>Broker:包括路由，并且管理，老的称谓( MOM )
<br>Message Broker:消息路由、消息管理(消息是否可达)
<br><br>6. 通讯协议( Protocol )
<br>Dubbo: Hession、Java Serialization (二进制)，跨语言不变，一般通过Client(Java、C++)
<br>二进制的性能是非常好：字节流，免去字符流(字符编码) ，机器友好、对人不友好
<br>序列化: 把编程语言数据结构转换成字节流、反序列化:字节流转换成编程语言的数据结构(原生类型的组合)
## 2、高可用架构
<br>1. 基本原则
<br>1) 消除单点失败
<br>2) 可靠性交迭
<br>3) 故障探测
<br><br>2. 可用性比率计算
<br>可用性比率:通过时间来计算(一年或者一月)
<br>比如:一年99.99%
<br>可用时间: 365 * 24 * 3600 * 99.99%
<br>不可用时间: 365 * 24 * 3600 * 0.01% = 3153.6秒 < 一个小时
<br>不可用时问: 1个小时推算一年 1 / 24 / 365 = 0.01 %
<br><br>3. 可靠性比率计算
<br>微服务里面的问题:
<br>一次调用:
<br>A->  B  -> C
<br>99%-> 99%-> 99%= 97%
<br>A->  B  -> C -> D
<br>99%-> 99%-> 99% -> 99% = 96%
<br><br>单台机器不可用比率:  1%
<br>两台机器不可用比率: 1% * 1%
<br>N机器不可用比率: 1%^ n
<br><br>结论:增加机器可以提高可用性，增加服务调用会降低可靠性，同时降低了可用性
## 3、Eureka
<br>1.  概念：服务发现是基于微服务的体系结构的关键原则之一。试图手动配置每个客户端或某种形式的约定可能很困难，而且很脆弱。
Eureka是Netflix服务发现服务器和客户端。可以将服务器配置和部署为高度可用，每个服务器将注册服务的状态复制到其他服务器。
<br><br> 2. Eureka服务器
<br><br>1) 关键代码
```properties
spring.application.name = spring-cloud-eureka-server
server.port = 9090
### 取消服务器自我注册
eureka.client.register-with-eureka = false
### 注册中心的服务器，没有必要再去检索服务
eureka.client.fetch-registry = false
### eureka 服务用于客户端注册
eureka.client.service-url.defaultZone = http://spring-cloud-eureka-server:${server.port}/eureka
### 多个： eureka.client.service-url.defaultZone =http://spring-cloud-eureka-server01:9091/eureka/,http://spring-cloud-eureka-server02:9092/eureka/
#eureka服务端的实例名称
eureka.instance.hostname = spring-cloud-eureka-server
```
```java
@SpringBootApplication
@EnableEurekaServer //激活
public class EurekaServerApplication {
	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}
}
```
<br><br>2) 通常经验
<br>Eureka服务器不需要开启自动注册，也不需要检索服务，但是这两个设置并不是影响作为服务器的使用，不过建议关闭，
为了减少不必要的异常堆栈，减少错误的干扰(比如:系统异常和业务异常)
<br><br>3) 为什么加`@EnableEurekaServer`才激活服务
这是一种常用设计方式
<br>Fast Fail:快速失败
<br>Fault-Tolerance :容错
<br><br>3.  Eureka客户端 Provider
关键代码
```properties
spring.application.name = user-service-provider
server.port=7070
### 用于客户端注册
eureka.client.service-url.defaultZone = http://spring-cloud-eureka-server:9090/eureka
eureka.instance.instance-id=user-service-provider
eureka.instance.prefer-ip-address=true
```
```java
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceProviderApplication {
	public static void main(String[] args) {
		SpringApplication.run(UserServiceProviderApplication.class, args);
	}
}
```
<br><br>4. Eureka客户端 Consumer
关键代码
```properties
spring.application.name = user-service-consumer
server.port = 8080
### 用于客户端注册
eureka.client.service-url.defaultZone = http://spring-cloud-eureka-server:9090/eureka
### 取消服务器自我注册
eureka.client.register-with-eureka = false
```
```java
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceConsumerApplication {
	public static void main(String[] args) {
		SpringApplication.run(UserServiceConsumerApplication.class, args);
	}
	@LoadBalanced
	@Bean
	public RestTemplate restTemplate(){
		return  new RestTemplate();
	}

}

/**
 * Created by yan on  29/09/2018.
 */
@Service
public class UserServiceProxy implements UserService {

    private static final String PROVIDER_SERVER_URL_PREFIX = "http://user-service-provider";

    //通过REST API代理到服务器提供者
    @Autowired
    private RestTemplate restTemplate;

    @Override
    public boolean sava(User user) {
        User returnValue = restTemplate.postForObject(PROVIDER_SERVER_URL_PREFIX + "/user/save", user, User.class);
        return returnValue != null;
    }

    @Override
    public Collection<User> findAll() {
        return restTemplate.getForObject(PROVIDER_SERVER_URL_PREFIX + "/user/list", Collection.class);
    }
}
```
## 4、思考
<br>1.  consul和Eureka是一样的吗?
<br>提供功能类似，Eureka 高可用，consul强一致性，consul功能更强大，广播式服务发现/注册
#### [差异](https://www.consul.io/intro/vs/eureka.html)    
<br>2.  重启eureka服务器，客户端应用要重启吗?
<br>不用，客户端在不停地上报信息，不过在Eureka服务器启动过程中，客户端大量报错
<br><br>3. 客户端上报的信息存储在哪里?
<br> 查看源码`DiscoveryClient`类中可以看到
```java
//初始化时，获取eureka-server所有注册表信息保存到本地
 private Applications filterAndShuffle(Applications apps) {
        if (apps != null) {
            if (this.isFetchingRemoteRegionRegistries()) {
                Map<String, Applications> remoteRegionVsApps = new ConcurrentHashMap();
                apps.shuffleAndIndexInstances(remoteRegionVsApps, this.clientConfig, this.instanceRegionChecker);
                Iterator var3 = remoteRegionVsApps.values().iterator();
                while(var3.hasNext()) {
                    Applications applications = (Applications)var3.next();
                    applications.shuffleInstances(this.clientConfig.shouldFilterOnlyUpInstances());
                }
                this.remoteRegionVsApps = remoteRegionVsApps;
            } else {
                apps.shuffleInstances(this.clientConfig.shouldFilterOnlyUpInstances());
            }
        }
        return apps;
    }
```
<br>都是在内存里面缓存着，EurekaClient 并不是所有的服务，存储需要的服务。比如: Eureka Server管理了200个应用，每个应用存在100个实例，总体管理20000个实例。客户端更具自己的需要的应用实例。
<br><br>4. consumer调用provider-a挂了，会自动切换provider-b吗，保证请求可用吗?
<br>当provider-a挂，会自动切换，不过不一定及时。不及时，服务端可能存在脏数据，或者轮训更新时间未达
<br><br>5.  一个业务中调用多个service时如何保证事务?
<br>需要分布式事务实现(JTA)，可是一般互联网项目，没有这种昂贵的操作