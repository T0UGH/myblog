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

### 11.4 使用`XML`配置开发`SpringAOP`

### 11.5 经典`SpringAOP`应用程序

### 11.6 多个切面

### 11.7 小结