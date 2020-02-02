---
title: '[SSM][11][面向切面编程]'
date: 2020-02-02 11:34:20
tags:
    - Spring
    - AOP
categories:
    - SSM
---
## 第 11 章 面向切面编程

### 11.1 一个简单的约定游戏

略

### 11.2 SpringAOP的基本概念

#### 11.2.1 AOP的概念和使用原因

先看一个不使用SpringAOP的例子，这里使用MyBatis框架，完成扣减一个产品的库存并且新增一笔交易记录的功能
````java
public void savePurchaseRecord(Long productId, PurchaseRecord record){
    SqlSession sqlSession = null;
    try{
        sqlSession = SqlSessionFactoryUtils.openSqlSession();
        ProductMapper productMapper = sqlSession.getMapper(ProductMapper.class);
        Product product = productMapper.getRole(productId);
        if(product.getStock() >= record.getQuantity()){
            product.setStock(product.getStock() - record.getQuantity());
            productMapper.update(product);
            PurchaseRecordMapper purchaseRecordMapper = sqlSession.getMapper(PurchaseRecordMapper.class);
            purchaseRecordMapper.save(record);
            sqlSession.commit();
        }
    }catch(Exception e){
        e.printStackTrace();
        sqlSession.rollBack();
    }finally{
        if(sqlSession != null){
            sqlSession.close();
        }
    }
}
````
- 这里的购买交易的产品和购买记录都在`try...catch...finally...`语句中
- 业务流程中穿插着事务的提交和回滚
- 并不是一个很好的设计

针对上一个例子的缺点，`SpringAOP`希望开发者能写成下面例子的代码
````java
@Autowired
private ProductMapper productMapper = null;
@Autowired
private PurchaseRecordMapper purchaseRecordMapper = null;
//.....
@Transactional
public void updateRoleNote(Long productId, PurchaseRecord record){
    Product product = productMapper.getRole(productId);
    if(product.getStock() >= record.getQuantity()){
        product.setStock(product.getStock() - record.getQuantity());
        productMapper.update(product);
        purchaseRecordMapper.save(record);
    }
}
````
- 这段代码除了一个注解`@Transactional`，没有任何打开、关闭数据库资源或者提交回滚事务的代码，却能完成与上面例子相同的功能
- 这段代码主要集中在业务处理上，而不是数据库事务和资源控制，这就是`AOP`的魅力

接下来我们探讨`SpringAOP`是如何做到这点的。

`AOP`可以通过动态代理模式，带来管控各个对象操作的切面环境，管理包括日志、数据库事务等操作，让我们拥有在反射原有对象方法之前、正常返回之后、异常返回之后插入自己代码的能力，有时我们甚至可以取代原有对象方法。

下面还是以数据库事务的例子来做说明
- 首先我们来了解以下正常`SQL`的逻辑步骤
    1. 通过数据库连接池来获得数据库连接资源，并进行一定的设置工作
    2. 执行对应的`SQL`语句，对数据进行操作
    3. 如果`SQL`执行过程中发生异常，回滚事务
    4. 如果`SQL`执行过程中没有发生异常，最后提交事务
    5. 关闭连接资源
- `SQL`逻辑步骤的流程图如下
    ![](src/200202_0.png)
- 而作为`AOP`，完全可以根据这个流程做一定的封装，然后通过动态代理技术，将代码织入到对应的流程中，我们完全可以进行如下设计
    1. 打开获取数据连接在`before`方法中完成
    2. 执行`SQL`，通过反射机制调用业务逻辑
    3. 如果`SQL`执行过程中发生异常，回滚事务，如果`SQL`执行过程中没有发生异常，提交事务，关闭连接资源

- 更重要的，对于数据库事务这种通用的操作来说，`SpringAOP`已经提供了一些通用的拦截器来处理，并不需要开发者自己实现。开发者只需要修改配置，就可以定制想要的功能。这就是`@Transactional`标签要做的，当方法标注为`@Transcational`标签时，方法启用数据库事务功能。如下图所示
    ![](src/200202_1.png)

- 通过这种方式，达成了约定优于配置的原则，使得开发者更关注于业务逻辑本身，而不是资源的控制。

#### 11.2.2 面向切面编程的术语

这一节来解释一些`AOP`的常用术语

##### 1. 切面

切面就是在一个怎么样的环境中工作，它可以定义后面需要介绍的通知、切点和引入等内容，然后`SpringAOP`会将其定义的内容织入到约定的流程中，在动态代理中可以把它理解为拦截器，比如类`RoleInterceptor`就是一个切面类。而常见的比如说，数据库事务就可以理解为一个切面

##### 2 通知

通知是切面的方法，与动态代理中，拦截器的方法类似，通知有如下种类:
- 前置通知(`before`): 在动态代理反射原有对象方法前执行的通知
- 后置通知(`after`): 在动态代理反射原有对象方法后执行的通知
- 返回通知(`afterRuturning`):在动态代理反射原有对象方法正常返回后执行的通知
- 异常通知(`afterThrowing`):在动态代理反射原有对象方法产生异常后执行的通知
- 环绕通知(`around`):它可以取代当前被拦截对象的方法

##### 3 引入

引入允许我们在现有的类里添加自定义的类和方法

##### 4 切点

用来告诉`SpringAOP`在什么时候启动拦截并织入对应的流程中

##### 5 连接点

连接点就是需要拦截器拦截的方法，比如例子中的`savePurchaseRecord`就是一个连接点

##### 6 织入

织入是一个生成代理对象并将切面内容放入到流程中的过程

AOP的流程图如下所示
![](src/200202_2.png)

#### 11.2.3 Spring对AOP的支持

`SpringAOP`是一种基于方法拦截的`AOP`。在`Spring`中有4种方法去实现`AOP`的拦截
- 使用`ProxyFactoryBean`和对应的接口实现`AOP`
- 使用`XML`配置`AOP`
- 使用`@AspectJ`注解驱动切面
- 使用`AspectJ`注入切面

### 11.3 使用`@AspectJ`注解开发`SpringAOP`

#### 11.3.1 选择连接点

`Spring`是方法级别的`AOP`框架，所以连接点只能是某个类下的某个方法。用动态代理来理解，就是要拦截哪个方法织入对应的`AOP`通知。

例子：我们先定义一个接口，和一个实现类，然后将实现类里的方法作为连接点
````java
package ComponentDemo;

public interface RoleService {
    public void printRoleInfo(Role role);
}
````

````java
package ComponentDemo;

import org.springframework.stereotype.Component;

@Component(value = "roleServiceImpl")
public class RoleServiceImpl implements RoleService {
    @Override
    public void printRoleInfo(Role role) {
        System.out.println("id = " + role.getId());
        System.out.println("roleName = " + role.getRoleName());
        System.out.println("note = " + role.getNote());
    }
}
````

#### 11.3.2 创建切面

在`Spring`中使用`@Aspect`注解一个类，那么这个类就会被看作切面，就相当于动态代理中的拦截器类

````java
package ComponentDemo;


import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Aspect
@Component(value = "roleAspect")
public class RoleAspect{
    @Before("execution(* ComponentDemo.RoleServiceImpl.printRoleInfo(..))")
    public void before(){
        System.out.println("before......");
    }

    @After("execution(* ComponentDemo.RoleServiceImpl.printRoleInfo(..))")
    public void after(){
        System.out.println("after......");
    }

    @AfterReturning("execution(* ComponentDemo.RoleServiceImpl.printRoleInfo(..))")
    public void afterReturning(){
        System.out.println("afterReturning......");
    }

    @AfterThrowing("execution(* ComponentDemo.RoleServiceImpl.printRoleInfo(..))")
    public void afterThrowing(){
        System.out.println("afterThrowing......");
    }
}
````

#### 11.3.3 定义切点

毕竟不是所有方法都需要使用`AOP`编程，所以我们的程序要有判断是否启用切面的功能，这也就是切点的定义，确定对于哪些调用启用切面。

如上面的例程所示，`Spring`是通过`@Before`这类注解后的正则表达式来判断切点的，例如下面这个表达式`execution(* ComponentDemo.RoleServiceImpl.printRoleInfo(..))`
- `execution`: 表示执行方法的时候触发
- `*`:代表任意返回类型的方法
- `ComponentDemo.RoleServiceImpl`: 代表类的全限定名
- `printRoleInfo`: 是被拦截的方法名称
- `(..)`: 任意的参数

进一步讨论这个正则表达式，它还可以配置如下内容

| AspectJ | 描述 |
|--|--|
|`arg()`|规定连接点匹配参数为指定类型的方法|
|`@args()`|规定连接点匹配指定注解标注的方法|
|`execution`|匹配连接点的执行方法|
|`this()`|规定连接点匹配`AOP`代理的`Bean`|
|`target`|规定连接点匹配被代理对象为指定的类型|
|`@target()`|规定连接点匹配特定的执行对象，这些对象要符合指定的注解类型|
|`within()`|规定连接点匹配指定的包|
|`@within()`|规定连接点匹配指定的类型|
|`@annotation`|规定匹配带有指定注解的连接点|

#### 11.3.4 测试AOP

这一节来编写程序测试`AOP`是否生效

首先需要进行`Spring` `Bean`的配置
````java
package ComponentDemo;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@EnableAspectJAutoProxy
@ComponentScan(basePackages = {"ComponentDemo"})
public class PojoConfig {
}
````
- `@EnableAspectJAutoProxy`代表启用`AspectJ`框架的自动代理，这时`Spring`才会生成动态代理对象，进而使用`AOP`

然后编写程序的主入口即可
````java
package ComponentDemo;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
public class Main {
    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(PojoConfig.class);
        Role role = ctx.getBean(Role.class);
        RoleService roleService = ctx.getBean(RoleService.class);
        roleService.printRoleInfo(role);
        ((AnnotationConfigApplicationContext) ctx).close();
    }
}
````

运行结果如下
````
before......
id = 1
roleName = role_name_1
note = role_note_1
after......
afterReturning......
````

#### 11.3.5 环绕通知

环绕通知是`SpringAOP`中最强大的通知，它可以同时实现前置通知和后置通知。它保留了调度被代理对象原有方法的功能，所以它既强大，又灵活。但是由于强大，它的可控制性不那么强，如果不需要大量改变业务逻辑，一般而言并不需要使用它。

例如
````java
 @Around("execution(* ComponentDemo.RoleServiceImpl.printRoleInfo(..))")
    public void around(ProceedingJoinPoint jp){
        System.out.println("around before......");
        try{
            jp.proceed();
        }catch(Throwable e){
            e.printStackTrace();
        }
        System.out.println("around after......");
    }
````

#### 11.3.6 织入

织入是生成代理对象并将切面内容放入约定流程的过程。使用`JDK`动态代理时，必须拥有接口，而使用`CGLib` 则不需要，于是`Spring`就提供了一个规则：当类的实现存在接口的时候，`Spring`将提供`JDK`动态代理。而当类不存在接口的时候没有办法使用`JDK`动态代理，`Spring`会采用`CGLIB`来生成代理对象。

#### 11.3.7 给通知传递参数

有时我们希望给各类通知传递参数，可以使用如下方法
````java
@Before("execution(* ComponentDemo.RoleServiceImpl.printRoleInfo(..))" + "&& args(role)")
public void before(Role role){
    System.out.println("before......" + role.getRoleName());
}
````
- 在切点的表达式中，加入了参数的定义，这样便可以传递参数

#### 11.3.8 引入

略

### 11.4 使用`XML`配置开发`SpringAOP`

方法大致和使用注解类似，这里不详细说明

### 11.5 经典`SpringAOP`应用程序

略

### 11.6 多个切面

Spring支持使多个切面按照指定的顺序运行，这时候可以使用注解`@Order`
````java
@Aspect
@Order(1)
public class Aspect1{
}
````
````java
@Aspect
@Order(2)
public class Aspect2{
}
````
````java
@Aspect
@Order(3)
public class Aspect2{
}
````

Spring底层是通过责任链来处理多个切面的，如下图
![](src/200202_3.png)


### 11.7 小结

`AOP`是`Spring`两大核心内容之一，通过`AOP`可以将一些比较公用的代码抽取出来，进而减少开发者的工作量