# Microservice

## 微服务

### 单体应用程序

- 数据库的表对所有模块可见
- 一个人的修改整个应用都要重新构建、测试、部署

### 微服务特征

- 应用程序分解为具有明确定义了职责范围的细粒度组件
- 完全独立部署，独立测试，并可复用
- 使用轻量级通信协议，HTTP和JSON，松耦合
- 服务实现可使用多种编程语言和技术
- 将大型团队划分成多个小型开发团队，每个团队只负责他们各自的服务

### Spring Boot和Spring Cloud

Spring Boot提供了基于java的、面向REST的微服务框架

Spring Cloud使实施和部署微服务到私有云或公有云变得更加简单（对第三方库进行集成）

#### 微服务要考虑的问题

- 微服务划分，服务粒度、通信协议、接口设计、配置管理、使用事件解耦微服务
- 服务注册、发现和路由
- 弹性，负载均衡，断路器模式（熔断），容错
- 可伸缩
- 日志记录和跟踪
- 安全
- 构建和部署，基础设施即代码

#### Rest原则

Representational State Transfer，表现层状态转移

- 资源（Resources），就是网络上的一个实体，标识：URI
- 表现层（Representation）：json、xml、html、pdf、excel
- 状态转移（State Transfer）：服务端--客户端

HTTP协议的四个操作方式的动词：GET、POST、PUT、DELETE

CRUD：`Create`（POST）、`Read`（GET）、`Update`（PUT）、`Delete`（DELETE）

如果一个架构符合REST原则，就称它为RESTful架构。

#### Spring Boot创建

@SpringBootApplication

+ 配置类@Configuration

+ @ComponentScan

@RestController

+ @Controller、@ResponseBody

+ 请求响应，JSON编解码（序列化）

mvn spring-boot:run

健康检查：localhost:8080/health（需要Actuator）

## 配置服务

### 为什么要分开

服务多，配置多，维护麻烦

方便使用多种数据源

### 配置信息和代码分开

配置信息硬编码到代码中 - 最低层级

分离的外部属性文件

与物理部署分离，如外部数据库

配置作为单独的服务提供（配置管理服务） - 最高层级

配置管理更改需要通知到使用数据的服务

### 配置方法

基于Spring Boot

pom.xml文件中的依赖

+ spring-cloud-config-server

+ spring-cloud-starter-config

指定父模块

导入dependencyManagement，依赖的构件和版本号

application.yml：服务端口号、后端数据来源

+ git、native

bootstrap.yml，指定服务名configserver

@EnableConfigServer

数据来源本地文件系统（ native）

#### 不同环境

`@Profile`

`-Dspring.profiles.active=***`

命名约定：应用程序名称-环境名称.yml，例如http://localhost:8888/licensingservice/default

数据来源Git

数据更新

### 属性注入

@Value：数据库属性自动创建数据源对象

@RefreshScope：属性刷新

### 端点

refresh端点：属性刷新

health端点：查看运行情况

env端点：获取全部环境属性

### 访问配置服务

#### 配置服务

```yaml
server:
  port: 8888
spring:
  profiles:
    active: native
  cloud:
    config:
      server:
        encrypt.enabled: true
        native:
          searchLocations: classpath:config/licensingservice
```

#### 获取配置

```yaml
spring:
  application:
    name: licensingservice
  profiles:
    active:
      default
  cloud:
    config:
      uri: http://localhost:8888
```

## 服务发现与负载均衡

### 服务发现的好处

快速水平伸缩（请求太多时，可以动态的增加容器），而不是垂直伸缩。不影响客户端（客户端不需要知道实例的数量）

提高应用程序的弹性（系统出错时的恢复能力，在某个服务出问题时，服务代理会将该服务删掉，客户端不会访问到）

### 服务调用关系

![服务调用关系](Microservice.assets/服务调用关系.png)

### Eureka

服务代理

多实例，为了解决并行请求、增加数据可靠性。

分为服务端和客户端，服务端进行服务的注册，微服务起来之后会向Eureka服务端进行注册，服务的客户端通过访问Eureka Server进行获取访问的端口号。客户端负责注册信息与接受访问服务。

![image-20210423233433078](Microservice.assets/Eureka结构.png)

#### Spring实现

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
#Default port is 8761
server:
  port: 8761

eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
  server:
    waitTimeInMsWhenSyncEmpty: 5 # 单位是毫秒，在服务启动时要来注册，要等待这个时间后服务才能被访问
  serviceUrl:
    defaultZone: http://localhost:8761
```

### Ribbon

集成在服务的进程中，解决客户端负载均衡的问题

主要提供客户侧的软件负载均衡算法

需要缓存Eureka Server的所有信息，当一个服务要被访问时，Ribbon按照一定的策略挑一个服务去调用一个服务给他。

![Ribbon架构](Microservice.assets/Ribbon架构.png)

#### 三种方式

+ Spring DiscoveryClient：@EnableDiscoveryClient

+ 使用支持Ribbon的RestTemplate

+ 使用Netflix Feign（简化客户端开发）

#### Spring DiscoveryClient

pom.xml文件中的依赖

+ spring-cloud-starter-eureka

启动类加@EnableDiscoveryClient，使能够使用DiscoveryClient和Ribbon库

注入：private DiscoveryClient discoveryClient;

discoveryClient.getInstances

new RestTemplate（访问rest API的方式，在这种方式是自己实现）

restTemplate.exchange（进行方法调用）

（没有使用ribbon的负载均衡能力。自己实例化RestTemplate，自己构建URL）

#### 支持Ribbon的RestTemplate

使用Ribbon功能

@LoadBalanced（加上Ribbon的负载均衡效果）

注入RestTemplate restTemplate;

restTemplate.exchange，指定要调用的服务名，而不是IP

#### Netflix Feign调用服务

pom.xml文件中的依赖

+ spring-cloud-starter-feign

启动类加注解：@EnableFeignClients

定义接口

+ 接口加注解：@FeignClient("organizationservice")
+ 不需要实现接口

### 自定义负载均衡策略

自己实现IRule接口就行

```java
@Bean
public IRule ribbonRule() {
    return new RandomRule();//BestAvailableRule();
}
```

## 客户端弹性

### 好处

远程服务发生错误或表现不佳导致的问题：客户端长时间等待调用返回

客户端弹性模式要解决的重点：让客户端免于崩溃。

目标：让客户端快速失败，而不消耗数据库连接或线程池之类的宝贵资源，防止远程服务的问题向客户端上游传播。（防止级联故障）

### 四种模式

#### 客户端负载均衡（client load banlance）模式

+ Ribbon提供的负载均衡器，帮助发现问题，并删除实例

#### 断路器模式（Circuit Breaker Patten）

+ 监视调用失败的次数，快速失败

#### 后备（fallback）模式

+ 远程服务调用失败，执行替代代码路径

#### 舱壁隔离模式（Bulkhead Isolation Pattern）

+ 线程池充当服务的舱壁

### Hystrix

Hystrix是一个延迟和容错库，旨在隔离对远程系统，服务和第三方库的访问点，停止级联故障，并在不可避免发生故障的复杂分布式系统中实现弹性。

pom.xml文件中的依赖

+ `spring-cloud-starter-hystrix`

启动类加注解：`@EnableCircuitBreaker`

用断路器包装远程资源调用，方法加注解：`@HystrixCommand`

默认1秒超时，超时会抛异常：`com.netflix.hystrix.exception.HystrixRuntimeException`

```java
@HystrixCommand(
     commandProperties = {@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "12000")}
 )
```

### 后备（fallback）模式

`fallbackMethod = "buildFallbackCargo"`

buildFallbackCargo方法位于相同类，与原方法具有相同签名

```java
@HystrixCommand(
     fallbackMethod = "buildFallbackCargo")
```

### 舱壁隔离模式（Bulkhead Isolation Pattern）

Hystrix默认共享同一个线程池（10个线程），用于不同的远程资源访问

```java
@HystrixCommand(
     threadPoolKey = "getCargoThreadPool",
     threadPoolProperties =
         {
             @HystrixProperty(name = "coreSize", value = "30"),
             @HystrixProperty(name = "maxQueueSize", value = "10")
         }
)
```

### 断路器模式(Circuit Breaker Patten)

![断路器模式](Microservice.assets/断路器模式.png)

## 服务网关

### 分布式系统的横切关注点

cross-cutting concern

- 安全
- 日志
- 用户跟踪
- 服务网关（service gateway）

### 服务网关

- 服务网关位于服务客户端和相应的服务实例之间
- 服务之间不能直接访问，要通过网关访问
- 所有服务调用（内部和外部）都应流经服务网关

服务网关提供的能力

+ 静态路由
+ 动态路由
+ 验证和授权
+ 度量数据收集和日志记录

### Zuul

将应用程序中的所有服务的路由映射到一个URL

网关在获得请求后，向服务代理去要地址，网关内也有ribbon负责进行负载均衡

![网关服务](Microservice.assets/网关服务.png)

#### 自动配置

服务ID

Zuul需要访问Eureka，查看注册的服务。有服务才会创建路由

#### 手动配置

静态URL是指向未通过Eureka服务发现引擎注册的服务的URL

禁用Ribbon与Eureka集成，手动指定负载均衡的服务实例

#### Spring实现

```yaml
#zuul.ignored-services: "*" # 忽略在建立时的自动静态配置
zuul.prefix:  /api # 前缀
zuul.routes.organizationservice: /organization/** # 手动静态配置
zuul.routes.licensingservice: /licensing/**
zuul.routes.authenticationservice: /auth/**

zuul.routes.licensestatic.path: /licensestatic/** # 外部不使用spring写的服务，根据静态路由转发到对应地址
zuul.routes.licensestatic.url:  http://licenseservice-static:8081
#zuul.routes.licensestatic.serviceId: licensestatic # 实现负载均衡
#zuul.routes.licensestatic.ribbon.listOfServers: http://licenseservice-static1:8081, http://licenseservice-static2:8082
#ribbon.eureka.enabled: false

zuul.sensitiveHeaders: Cookie,Set-Cookie
zuul.debug.request: true
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 2500
#hystrix.command.licensingservice.execution.isolation.thread.timeoutInMilliseconds: 2
#licensingservice.ribbon.ReadTimeout: 2
signing.key: "345345fsdfsf5345"

```

#### 动态重新加载

Git-更新zuulservice配置

zuul POST:http://localhost:5555/refresh

#### 设置超时时间

Hystrix,1S

Ribbon,5S

Ribbon的懒加载导致第一次调用慢，引起失败

### Zuul过滤器

使用Zuul和Zuul过滤器允许开发人员为通过Zuul路由的所有服务实现横切关注点

ZuulFilter

+ `前置过滤器`，在Zuul将实际请求发送到目的地之前被调用
+ `后置过滤器`，在目标服务被调用并将响应发送回客户端后被调用
+ `路由过滤器`，用于在调用目标服务之前拦截调用

![image-20210424013104887](Microservice.assets/Zuul过滤器.png)

#### 关联ID

Zuul会在请求进来时，加上一个关联ID，并且将请求头部存在Zuul的一个内存区域代为管理。通过请求的关联ID，在每一次请求的时候，可以根据ID知道这是同一次请求的处理。

关联ID在每一个服务的RestTemplate的处理时将关联ID附上

#### 前置过滤器

```java
public Object run() {

    if (isCorrelationIdPresent()) {
       logger.debug("tmx-correlation-id found in tracking filter: {}. ", filterUtils.getCorrelationId());
    }
    else{
        filterUtils.setCorrelationId(generateCorrelationId());
        logger.debug("tmx-correlation-id generated in tracking filter: {}.", filterUtils.getCorrelationId());
    }

    RequestContext ctx = RequestContext.getCurrentContext(); // 获取请求上下文
    logger.debug("Processing incoming request for {}.",  ctx.getRequest().getRequestURI());
    logger.debug("====incoming=========ServiceId=={}",filterUtils.getServiceId());
    return null;
}
```

#### 后置过滤器

```java
@Override
public Object run() {
    RequestContext ctx = RequestContext.getCurrentContext();

    logger.debug("Adding the correlation id to the outbound headers. {}", filterUtils.getCorrelationId());
    ctx.getResponse().addHeader(FilterUtils.CORRELATION_ID, filterUtils.getCorrelationId()); // 附上内存中存着的关联ID

    logger.debug("Completing outgoing request for {}.", ctx.getRequest().getRequestURI());
    logger.debug("====outgoing=========ServiceId=={}",filterUtils.getServiceId());
    return null;
}
```

#### 路由过滤器

动态路由，重新构造一个request的请求，并且决定要发往哪里。（太复杂了老师也没实现）

### Zuul和Eureka和Ribbon和Feign（重要）

Zuul是总的入口，他要向Eureka获取当前系统当中注册的服务以及状态，针对up的服务建立相应的路由，比如静态路由的方式。由Zuul代为访问目标服务，访问目标服务的方式由Ribbon决定，Ribbon来做负载均衡。Feign简化了客户端访问服务的方式，只需要透过Feign接口就能访问服务，但是还是要借助Ribbon去访问，也就是每一个服务都要透过Ribbon进行负载均衡。