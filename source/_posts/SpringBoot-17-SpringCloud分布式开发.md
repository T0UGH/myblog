---
title: '[SpringBoot][17][SpringCloud分布式开发]'
date: 2020-02-17 14:43:43
tags:
    - SpringCloud
    - SpringBoot
categories:
    - SpringBoot
---

## 第 17 章 SpringCloud分布式开发

按照现今互联网的开发，高并发、大数据、快响应已经是普遍的要求。为了支撑这样的需求，互联网系统也开始引入分布式的开发。为了实现分布式的开发， Spring推出了一套组件，那就是 Spring Cloud。它将目前各家公司已经开发好的、经过实践考验较为成熟的技术组合起来，并且通过 Spring Boot风格进行再次封装，从而屏蔽掉了复杂的配置和实现原理，为开发者提供了一套简单易懂、易部署和维护的分布式系统开发包。

Spring Cloud是一套组件，可以细分为多种组件，如服务发现、配置中心、消息总线、负载均衡、断路器和数据监控等。

- 服务治理和服务发现
    - 在 Spring Cloud中主要是使用 Netflix Eureka作为服务治理的
    - 通过服务注册将单个微服务节点注册给服务治理中心，这样服务治理中心就可以治理单个微服务节点
    - 服务发现则是微服务节点可以对服务治理中心发送消息，使得服务治理中心可以将新的微服务节点纳入管理。

- 客户端负载均衡
    - 在微服务的开发中，会将一个大的系统拆分为多个微服务系统，而各个微服务系统之间需要相互协作才能完成业务需求。每一个微服务系统可能存在多个节点，当个微服务（服务消费者）调用另外一个微服务（服务提供者）时，服务提供者需要负载均衡算法提供一个节点进行响应。
    - 除此之外，在服务的过程中，可能出现某个节点故障的风险，通过均衡负载的算法就可以将故障节点排除，使后续请求分散到其他可用节点上
    - Spring Cloud为此提供了 Ribbon 来实现这些功能，主要使用的就是 RestTemplate

- 声明服务调用
    - 对于REST风格的调用，如果使用 RestTemplate会比较烦琐，可读性不高
    - 为了简化多次调用的复杂度， Spring Cloud提供了接口式的声明服务调用编程，它就是 Feign
    - 通过它请求其他微服务时，就如同调度本地服务的Java接口一样，从而提高代码的可读性

- 断路器
    - 在分布式中，因为存在网络延迟或者故障，所以一些服务调用无法及时响应。如果此时服务消费者还在大量地调用这些网络延迟或者故障的服务提供者，那么很快消费者也会因为大量的等待，造成积压，最终导致其自身出现服务瘫痪。
    - 为了克服这个问题， Spring Cloud引入了Netⅸx的开源框架 Hystix来处理这些问题。当服务提供者响应延迟或者故障时，就会使得服务消费者长期得不到响应， Hystix断路器就会对这些延迟或者故障的服务进行
    - 这样，当服务消费者长期得不到服务提供者响应时，就可以进行降级、服务断路、线程和信号隔离、请求缓存或者合并等处理

- API网关
    - 在 Spring Cloud中API网关是Zuul。对于网关而言，存在两个作用。
    - 第一个作用是将请求的地址映射为真实服务器的地址，显然这个作用就起到路由分发的作用，从而降低单个节点的负载。
    - Zuul网关的第二个作用是过滤服务，在互联网中，服务器可能面临各种攻击，Zuul提供了过滤器，通过它过滤那些恶意或者无效的请求，把它们排除在服务网站之外，这样就可以降低网站服务的风险。


为了更好的理解各个组件，我们举个例子来说明
![](SpringBoot-17-SpringCloud分布式开发\200215_7.png)
假设需要实现一个电商项目，当前团队需要承担两个模块的开发，分别是用户模块和产品模块。根据微服务的特点，将系统拆分为用户服务和产品服务，而两个服务通过REST风格请求进行交互。在分布式的环境下，为了提高处理能力、分摊单个系统的压力以及高可用的要求，往往需要一个微服务拥有两个或者以上的节点，如图17-1所示的架构。

### 17.1 服务治理和服务发现：Eureka

#### 17.1.1 配置服务治理节点

Spring Cloud的服务治理是使用 Netflix的 Eureka作为服务治理器的，它是构建 Spring Cloud 分布式最为核心和最为基础的模块，它的作用是注册和发现各个 Spring Boot 微服务，并且提供监控和治理功能

下面演示如何搭建一个服务注册中心

首先在`pom.xml`中引入依赖
````xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
        <scope>provided</scope>
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
````

这样就引入了 Eureka 模块的包。然而要启用它只需要在 Spring boot 的启动文件上加入注解`@EnableEurekaServer`便可以了

````java
@SpringBootApplication
@EnableEurekaServer
public class Chapter17ServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(Chapter17ServerApplication.class, args);
	}
}
````

我们还需要使用 `application.properties`进一步配置 Eureka 模块的一些基本内容
````
#Spring项目名称
spring.application.name=server
#服务器端口
server.port=7001
# Eureka注册服务器名称
eureka.instance.hostname=localhost
#是否注册给服务中心
eureka.client.register-with-eureka=false
#是否检索服务
eureka.client.fetch-registry=false
# 治理客户端服务域
eureka.client.serviceuUrl.defaultZone=http://localhost:7001/eureka/
````
- 属性`spring application.name`配置为`server`，这是一个标识，它表示某个微服务的共同标识。如果有第二个微服务节点启动时，也是将这个配置设为`server`，那么 Spring Cloud就会认为它也是`server`这个微服务的一个节点。
- 属性`eureka.client.register-with-eureka`配置为`false`，是因为在默认的情况下，项目会自动地查找服务治理中心去注册。这个工程自身就是服务治理中心，所以取消掉注册服务中心。
- 属性`eureka.client.fetch-registry`配置为`false`，它是一个检索服务的功能，因为服务治理中心是维护服务实例的，所以也不需要这个功能，即设置为了`false`
- 属性`eureka.client.serviceUrl.defaultZone`代表服务中心的域，将来可以提供给别的微服务注册。后面的微服务还会使用到它。

然后启动服务，并使用浏览器访问它
![](SpringBoot-17-SpringCloud分布式开发\200215_8.png)

#### 17.1.2 服务发现

下面我们新建一个产品微服务，然后让`Eureka`去发现它

首先在`pom.xml`中引入`Eureka`客户端包
````xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
....
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
````

然后使用`application.properties`来配置具体注册到哪个服务治理中心
````
#服务器端口
server.port=9001
#Spring服务名称
spring.application.name=product
#治理客户端服务域
eureka.client.serviceUrl.defaultZone=http://localhost:7001/eureka/
````
- 治理客户端服务域则是通过属性`eureka.client.serviceUrl.defaultZone`进行配置的，它也配置了服务治理中心同样的地址，这样它就能够注册到之前所配置的服务治理中心。

接着我们再使用上面的步骤创建一个用户微服务

#### 17.1.3 配置多个服务治理中心节点

我们现在希望存在两个服务治理中心节点，因为在服务治理中心也可能单个节点出现故障，导致服务不可用。

这时，我们首先修改上面`server`工程的配置文件，将这个服务的服务注册中心改为7002端口
````
#Spring应用名称
spring.application.name=server
#端口
server port=7001
#服务治理中心名称
eureka.instance.hostname=localhost
#将当前服务治理中心注册到7002端口的服务治理中
eureka.client.serviceUrl.defaultZone=http://localhost:7002/eureka/
````

接下来我们将这个工程整个复制，修改配置文件如下
````
#Spring应用名称
spring.application.name=server
#端口
server port=7002
#服务治理中心名称
eureka.instance.hostname=localhost
#将当前服务治理中心注册到7002端口的服务治理中
eureka.client.serviceUrl.defaultZone=http://localhost:7001/eureka/
````

这里可以看到两个服务治理中心是通过相互注册来保持相互监控的，关键点是属性`spring.application.name`保持一致都为`server`，这样就可以形成两个甚至是多个服务治理中心

接下来，需要考虑的是如何将其他的微服务注册到多个服务治理中心中。下面以单个产品微服务为例来修改配置文件
````
#服务器端口
server.port=9001
#Spring服务名称
spring.application.name=product
#注册多个治理客户端服务域
eureka.client.serviceurl.defaultzone=http://localhost7001/eureka/, http://localhost:7002/eureka/
````

### 17.2 微服务之间的调用

上面已经把产品和用户两个微服务注册到服务治理中心了。对于业务，则往往需要各个微服务之间相互地协助才能完成。为了方便从其他微服务中获取信息，生产者微服务会以`REST`风格提供一个请求`URL`。这样对于消费者微服务就可以通过`REST`请求获取用户服务。

除了处理获取其他服务的数据外，这里还需要注意服务节点之间的负载均衡。Spring Cloud提供了`Ribbon`和`Feign`组件来帮助我们完成这些功能。通过它们，各个微服务之间就能够相互调用，并且它会默认实现了负载均衡。

#### 17.2.1 Ribbon客户端负载均衡

假如有多个生产者服务，消费者应该访问哪个呢？这就是客户端负载均衡。而在SpringCloud体系中，客户端负载均衡时通过Ribbon实现的。对于Ribbon，它实际就是一个 RestTemplate对象。Spring Cloud提供了一个简单的`@LoadBalance`注解以实现客户端的负载均衡负载均衡的算法。

下面还是以用户微服务和产品微服务举例。假设此时用户微服务已经开发好了一个REST风格的接口，且在Eureka中注册了两个实例，这时，产品微服务要调用用户微服务，只要进行如下步骤就可以实现负载均衡了

首先加入对 Ribbon 的依赖
````xml
<dependency>
    <groupId>org. springframework cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
````

然后为RestTemplate加入负载均衡标签
````java
@SpringCloudApplication
public class Chapter17ProductApplication {

    // 初始化RestTemplate
    @LoadBalanced // 多节点负载均衡
    @Bean(name = "restTemplate")
    public RestTemplate initRestTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(Chapter17ProductApplication.class, args);
    }
}
````

这段代码中在`RestTemplate`上加入了注解`@LoadBalanced`。它的作用是让`RestTemplate`实现负载均衡，也就是，通过这个`RestTemplate`对象调用用户微服务请求的时候，`Ribbon`会自动给用户微服务节点实现负载均衡，这样请求就会被分摊到微服务的各个节点上，从而降低单点的压力。

为了测试在产品微服务中新建产品控制器，通过`RestTemplate`进行调用即可
````java
// 注入RestTemplate
@Autowired
private RestTemplate restTemplate = null;

@GetMapping("/ribbon")
public UserPo testRibbon() {
    UserPo user = null;
    // 循环10次，然后可以看到各个用户微服务后台的日志打印
    for (int i = 0; i < 10; i++) {
        // 注意这里直接使用了USER这个服务ID，代表用户微服务系统
        // 该ID通过属性spring.application.name来指定
        user = restTemplate.getForObject("http://USER/user/" + (i + 1), UserPo.class);
    }
    return user;
}
````
代码中注入了`RestTemplate`对象，这是自动实现客户端均衡负载的对象。然后在方法中使用`USER`这个字符串代替了服务器及其端口，这是一个服务ID，在Eureka服务器中可以看到它的各个节点，它是用户微服务通过属性 `spring.application.name`来指定的。

#### 17.2.2 Feign声明式调用

上节中使用了`RestTemplate`，但是使用`RestTemplate`有些麻烦。除了要编写URL，还需要注意这些参数的组装和结果的返回等操作。为了克服这些不友好，除了Ribbon外，Spring Cloud还提供了声明式调用组件Feign。

Feign是一个基于接口的编程方式，开发者只需要声明接口和配置注解，在调度接口方法时Spring Cloud就根据配置来调度对应的REST风格的请求，从其他微服务系统中获取数据。

我们举例来说明，首先引入依赖包
````xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
````

然后需要在 Spring Boot 的启动文件中加入注解`@EnableFeignClients`，这个注解代表该项目会启动 Feign客户端。

````java
@EnableFeignClients(basePackages = "com.springboot.chapter17.product")
@ComponentScan(basePackages = "com.springboot.chapter17.product")
@SpringCloudApplication
public class Chapter17ProductApplication {
}
````

然后在产品微服务中加入接口声明，注意这里仅仅是一个接口声明，并不需要实现类
````java
/**** imports ****/
// 指定服务ID（Service ID）
@FeignClient("user")
public interface UserService {
	// 指定通过HTTP的GET方法请求路径
	@GetMapping("/user/{id}")
	// 这里会采用Spring MVC的注解配置
	public UserPo getUser(@PathVariable("id") Long id);

	// POST方法请求用户微服务
	@PostMapping("/insert")
	public Map<String, Object> addUser(
			// 请求体参数
			@RequestBody UserPo user);

	// POST方法请求用户微服务
	@PostMapping("/update/{userName}")
	public Map<String, Object> updateName(
			// URL参数
			@PathVariable("userName") String userName,
			// 请求头参数
			@RequestHeader("id") Long id);

	// 调用用户微服务的timeout请求
	@GetMapping("/timeout")
	public String testTimeout();
}
````
`@FeignClient("user")`代表这是一个Feign客户端配置的`user`是一个服务的ID，它指向了用户微服务，这样Feign就会知道向用户微服务请求，并会实现负载均衡。这里的注解`@GetMapping`代表启用HTTP的GET请求用户微服务，而方法中的注解`@PathVariable`代表从URL中获取参数，这显然还是Spring MVC的规则，Spring Cloud之所以选择这样的方式，是为了降低读者的学习成本。然后通过IOC技术，这个`service`会在spring容器中产生一个可供调用实例。

下一步，在`Controller`中就可以像调用本地的`Service`一样去调用它了
````java
// 注入Feign接口
@Autowired
private UserService userService = null;

// 测试
@GetMapping("/feign")
public UserPo testFeign() {
    UserPo user = null;
    // 循环10次
    for (int i = 0; i < 10; i++) {
        Long id = (long) (i + 1);
        user = userService.getUser(id);
    }
    return user;
}
````

与Ribbon相比，Feign屏蔽掉了`RestTemplate`的使用，提供了接口声明式的调用，使得程序可读性更高，同时在多次调用中更为方便。

### 17.3 Hystrix断路器

在互联网中，可能存在某一个微服务的某个时刻压力变大导致服务缓慢，甚至出现故障，导致服务不能响应。这里假设用户微服务请求中出现压力过大，服务响应速度变缓，进入瘫痪状态，而这时产品微服务响应还是正常响应。但是如果出现产品微服务大量调用用户微服务，就会出现大量的等待，如果还是持续地调用，则会造成大量请求的积压，导致产品微服务最终也不可用。这种现象也称为雪崩

![](SpringBoot-17-SpringCloud分布式开发\200217_0.png)

为了防止这样的蔓延，微服务提出了断路器的概念。断路器就如同电路中的保险丝，如果电器耗电大，导致电流过大，那么保险丝就会熔断，从而保证用电的安全。同样地，在微服务系统之间大量调用可能导致服务消费者自身出现瘫痪的情况下，断路器就会将这些积压的大量请求“熔断”，来保证其自身服务可用，而不会蔓延到其他微服务系统上。通过这样的断路机制可以保持各个微服务持续可用。

处理限制请求的方式的策略很多，如限流、缓存等。这里主要介绍最为常用的降级服务。所谓降级服务，就是当请求其他微服务出现超时（timeout）或者发生故障时，就会使用自身服务其他的方法进行响应。

下面举例说明，这里首先在用户微服务中新增REST端点用来模拟超时
````java
@GetMapping("/timeout")
public String timeout() {
    // 生成一个3000之内的随机数
    long ms = (long)(3000L*Math.random());
    try {
        // 程序延迟，有一定的概率超过2000毫秒
        Thread.sleep(ms);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "熔断测试";
}
````
这个方法没有任何业务含义，只是会使用`sleep`方法让当前线程休眠随机的毫秒数。这个毫杪数可能超过2000mns，也就是有可能超过 Hystrix 所默认的2000ms的时间，这样就可以出现短路，进入降级方法。

接下来我们在产品微服务中启用断路器，首先需要在`maven`中引入它，然后在启动文件中加入`@EnableCircuitBreaker`就可以启动断路机制
````xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
````
````java
// 启动断路器
@EnableCircuitBreaker
//自定义扫描包
@ComponentScan(basePackages = "com.springboot.chapter17.product")
//开启Spring Boot应用、服务发现和断路器功能
@SpringCloudApplication
public class Chapter17ProductApplication {
    //.....
}
````

然后我们分别配置产品微服务里的`service`和`controller`就可以开始测试了
````java
// 调用用户微服务的timeout请求
@GetMapping("/timeout")
public String testTimeout();
````
````java
// Feign断路测试
@GetMapping("/circuitBreaker2")
@HystrixCommand(fallbackMethod = "error")
public String circuitBreaker2() {
    return userService.testTimeout();
}

// 降级服务方法
public String error() {
    return "超时出错。";
}
````

首先看到`@HystrixCommand`注解，它表示将在方法上启用断路机制，而其属性`fallbackMethod`则可以指定降级方法，指定为`error`，那么降级方法就是`error`。如此指定后，在请求`circuitBreaker2`时，只要超时过了2000ms，服务就会启用`error`方法作用响应请求，从而避免请求的积压，保证微服务的高可用性。

其流程如下图所示
![](SpringBoot-17-SpringCloud分布式开发\200217_1.png)

#### 17.3.2启用Hystrix仪表盘
对于 Hystrix, Spring Cloud还提供了一个仪表盘（Dashboard）进行监控断路的情况，从而让开发者监控可能出现的问题。

这里直接贴出配置过程的代码，新建一个工程简单配置它即可

````xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
````

````java
/****imports****/
@SpringBootApplication
// 启用Hystrix仪表盘
@EnableHystrixDashboard
public class Chapter17DashboardApplication {
	public static void main(String[] args) {
		SpringApplication.run(Chapter17DashboardApplication.class, args);
	}
}
````

````
server.port=6001
spring.application.name=hystrix_dashboard
````

具体如何使用略

### 17.4 Zuul路由网关

网关的功能对于分布式网站是十分重要的
- 首先它可以将请求路由到真实的服务器上，进而保护真实服务器的IP地址，避免直接地攻击真实服务器
- 其次它也可以作为一种负载均衡的手段，使得请求按照一定的算法平摊到多个节点上，减缓单点的压力
- 最后它还能提供过滤器，过滤器可以判定请求是否为有效请求，一旦判定失败，就可以将请求阻止，避免发送到真实的服务器。

#### 17.4.1 构建Zuul网关

下面新建一个工程，作为`Zuul`网关

````xml
<!--引入服务发现 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
````

````java
@SpringBootApplication(scanBasePackages = "com.springboot.chapter17.zuul")
//启动Zuul代理功能
@EnableZuulProxy
public class Chapter17ZuulApplication {

	public static void main(String[] args) {
		SpringApplication.run(Chapter17ZuulApplication.class, args);
	}
}
````

````
server.port=80
spring.application.name=zuul

zuul.routes.user-service.path=/u/**
zuul.routes.user-service.url=http://localhost:8001/

zuul.routes.product-service.path=/p/**
zuul.routes.product-service.serviceId=product

eureka.client.serviceUrl.defaultZone=http://localhost:7001/eureka/,http://localhost:7002/eureka/
````
- 这里引入了服务发现的包，所以我们也可以参照之前的论述将Zuul网关服务注册到服务治理中心去。
- Zuul已经引入了断路机制，之所以引入断路机制，是因为在请求不到的时候，会进行断路，以避免网关发生请求无法释放的场景，导致微服务瘫痪。
- 对于用户微服务映射的配置，这里采用 `zuul.routes.<key>.path`和 `zuul.routes.<key>.serviceId`进行配置，其中`path`是请求路径，这里使用了ANT风格的通配`/u/**`，而serviceId代表服务名称，这样配置后，zuul还会自动实现网关负载均衡，当请求到来时，会均衡请求到名字相同的服务集合上。
- 这时，当我们请求`http://localhost/p/product/ribbon`时，`http://localhost/`代表zuul网关的地址，`p`是我们刚才配置的转发，也就是product微服务的某个节点，而`/product/ribbon`则由product微服务再进一步解析寻址

#### 17.4.2 使用过滤器

有时候还希望网关功能更强大些。例如，监测用户登录、黑名单用户、购物验证码、恶意刷请求攻击等场景。如果这些在过滤器内判断失败，那么就不要再把请求转发到其他微服务上，以保护微服务的稳定。

具体实现略

### 17.5 使用@SpringCloudApplication

上面的内容中，对于启动文件采用了很多注解，如`@SpringBootApplication`、`@EnableDiscoveryClient`和`@EnableCircuitBreaker`等。这些注解有时候会让人觉得比较冗余，为了简化开发， Spring Cloud还提供了自己的注解`@SpringCloudApplication`来简化使用 Spring Cloud的开发。

使用了`@SpringCloudApplication`这个注解就相当于使用了`@SpringBootApplication`、`EnableDiscoveryClient`和`EnableCircuitBreaker`三个注解。

前面的产品微服务的启动文件就可以改为如下
````java
@EnableFeignClients(basePackages="com.springboot.chapter17.product")
@ComponentScan(basePackages="com.springboot.charter17.product")
@SpringCloudApplication
public class Chapter17ProductApplication {

	public static void main(String[] args) {
		SpringApplication.run(Chapter17ZuulApplication.class, args);
	}
}
````
- 因为`@SpringCloudApplication`并不提供配置包的功能，所以使用`@ComponentScan`进行包扫描来弥补
- `@SpringCloudApplication`并不会自动启用 Feign，所以使用`@EnableFeignCLients`来启用 Feign
