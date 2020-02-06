---
title: '[SpringBoot][1][SpringBoot的来临]'
date: 2020-02-06 19:00:43
tags:
    - SpringBoot
categories:
    - SpringBoot
---
## 第 1 章 `SpringBoot`的来临

### 1.1 `Spring`的历史

在`Spring`框架没有开发出来时，`JavaEE`是以`Sun`公司所制定的`EJB`(Enterprise Java Bean)作为标准的。

然后在`2004`年由`RodJohnson`主导的`Spring`项目推出了`1.0`版本，这彻底地改变了`JavaEE`开发的世界，很快人们就抛弃了繁重的`EJB`的标准，迅速地投入到了`Spring`框架中， 于是`Spring`成为了现实中`JavaEE`开发的标准。`Spring`以强大的控制反转(`loC`)来管理各类`Java`资源，从而降低了各种资源的藕合；并且提供了极低的侵入性，也就是使用`Spring`框架开发的编码，脱离了`Spring`也可以继续使用；而`Spring`的面向切面的编程(`AOP`)通过动态代理技术，允许我们按照约定进行配置编程，进而增强了`Bean`的功能，它擦除了大量重复的代码使得开发人员能够更加集中精力于业务开发；`Spring`还提供许多整合了当时非常流行的框架的模板，如持久层`Hibernate`的`HibernateTemplate`模板、`IBATIS`的`SqlMapClientTemplate`模板等， 极大地融合并简化了当时主流技术的使用， 使得其展示了强有力的生命力， 并延续至今。

### 1.2 注解还是`XML`

只是在`Spring`早期的`l.x`版本中，由于当时的`JDK`并不能支持注解，因此只能使用`XML`。而很快随着`JDK` 升级到`JDK5`，它加入了注解的新特性，这样注解就被广泛地使用起来，于是`Spring`的内部也分为了两派，一派是使用XML 的赞同派，一派是使用注解的赞同派。

为了简化开发，在`Spring2.x`之后的版本也引入了注解，不过只是少量的注解，如`@Component`、`@Service` 等，但是功能还不够强大，因此对于`Spring`的开发，绝大部分的情况下还是以使用`XML`为主， 注解为辅。

到了`Spring 3.0`后，引入了更多的注解功能，于是在`Spring`中产生了这样一个很大的分歧，即是使用注解还是使用XML？对`XML`的引入，有些人觉得过于繁复，而对于注解的使用，会使得注解分布得到处都是，难以控制，有时候还需要了解很多框架的内部实现才能准确使用注解开发所需的功能。这个时候大家形成了这样的一个不成文的共识，对于业务类使用注解，例如，对于`MVC`开发，控制器使用`@Controller`，业务层使用`@Service`，持久层使用`@Repository`；而对于一些公用的`Bean`，例如，对于数据库(如`Redis`)、第三方资源等则使用`XML`进行配置，直至今时今日这样的配置方式还在企业中广泛地使用着。也许使用注解还是`XML`是一个长期存在的话题，但是无论如何都有道理。

随着注解的功能增强，尤其是`Servlet 3.0`规范的提出，`Web`容器可以脱离`web.xml`的部署，使得`Web`容器完全可以基于注解开发，对于`Spring 3.x`和`Spring 4.x`的版本注解功能越来越强大，对于`XML`的依赖越来越少，到了`4.x`的版本后甚至可以完全脱离`XML`，因此在`Spring`中使用注解开发占据了主流的地位。与此同时，`Pivotal`团队在原有`Spring`的基础上主要通过注解的方式继续简化了`Spring`框架的开发，它们基于`Spring`框架开发了`Spring Boot`，所以`Spring Boot`并非是代替`Spring`框架，而是让`Spring`框架更加容易得到快速的使用。`Pivotal`团队在2014年推出`Spring Boot`的`1.0`版本，该版本使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。在2018年3月`SpringBoot`推出了`2.0.0 GA`版本，该版本是基于`Spring5`的，并引入其最新的功能，能够有效支持`Java 9`的开发。`Spring Boot`致力于在蓬勃发展的快速应用开发领域(rapid application development)借助`Java EE`在企业互联网的强势地位成为业界领导者，它也是近年来`Java`开发最令人感到惊喜的项目之一。

### 1.3 `SpringBoot`的优点

`SpringBoot`的优点
- 创建独立的`Spring`应用程序
- 嵌入的`Tomcat`、`Jetty`或者`Undertow`，无须部署`WAR`文件
- 允许通过`Maven`来根据需要获取`starter`
- 尽可能地自动配置`Spring`
- 提供生产就绪型功能，如指标、健康检查和外部配置
- 绝对没有代码生成，对`XML`没有要求配置

约定优于配置，这是`SpringBoot`的主导思想。对于`SpringBoot`而言，大部分情况下存在默认配置，你甚至可以在没有任何定义的情况下使用`Spring`框架，如果需要自定义也只需要在配置文件配置一些属性便可以，十分便捷。而对于部署这些项目必需的功能，`SpringBoot`提供`starter`的依赖，如`spring-boot-starter-web` 捆绑了`SpringMVC`所依赖的包，`spring-boot-starter-tomcat`绑定了内嵌的`Tomcat`，这样使得开发者能够尽可能快地搭建开发环境，快速进行开发和部署，这就是`SpringBoot`的特色。

### 1.4 传统`SpringMVC`和`SpringBoot`的对比

与传统的`SpringMVC`对比，`SpringBoot`允许直接进行开发，这就是它的优势。在传统所需要配置的地方， `SpringBoot`都进行了约定，也就是你可以直接以`SpringBoot`约定的方式进行开发和运行你的项目。当你需要修改配置的时候，它也提供了一些快速配置的约定，犹如它所承诺的那样，尽可能地配置好`Spring`项目和绑定对应的服务器，使得开发人员的配置更少，更加直接地开发项目。对于那些微服务而言，更喜欢的就是这样能够快速搭建环境的项目，而`SpringBoot`提供了这种可能性，同时`SpringBoot`还提供了监控的功能，随着云技术的
到来，微服务成了市场的热点，于是代表`Java`微服务时代的`SpringBoot`微服务开发的时代己经到来，结合`SpringCloud`后它还能很方便地构建分布式系统开发，满足大部分无能力单独开发分布式架构的企业所需，所以这无疑是激动人心的技术。

