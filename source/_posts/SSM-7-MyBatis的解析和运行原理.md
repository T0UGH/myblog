---
title: '[SSM][7][MyBatis的解析和运行原理]'
date: 2020-01-31 16:25:37
tags:
    - MyBatis
categories:
    - SSM
---
## 第 7 章 MyBatis的解析和运行原理

MyBatis的运行过程分为两大步
1. 读取配置文件缓存到`Configuration`对象，用以创建`SqlSessionFactory`
2. `SqlSession`的执行过程

### 7.1 构建SqlSessionFactory过程

`SqlSessionFactory`是`MyBatis`的核心类之一，其最重要的功能就是提供创建`MyBatis`的核心接口`SqlSession`，所以要先创建`SqlSessionFactory`，为此要提供配置文件和相关的参数。

`MyBatis`采用了`Builder`模式去创建`SqlSessionFactory`，在实际中可以通过`SqlSessionF actoryBuilder`去构建，其构建分为两步。
1. 通过`XMLConfigBuilder`解析配置的`XML`文件，读出所配置的参数，并将读取的内容存入`Configuration` 类对象中。
2. 使用`Confinguration` 对象去创建`SqlSessionFactory`

#### 7.1.1 构建Configuration

在`SqlSessionFactory`构建中，`Configuration`是最重要的，它的作用是：
- 读入配置文件，包括基础配置的`XML`和映射器`XML`(或注解)。
- 初始化一些基础配置，比如`MyBatis`的别名等，一些重要的类对象(比如插件、映
射器、`Object`工厂、`typeHandlers`对象等)。
- 提供单例，为后续创建`SessionFactorγ`服务，提供配置的参数。
- 执行一些重要对象的初始化方法。

`Configuration`是通过`XMLConfigBuilder`去构建的，首先它会读出所有`XML`配置的信息，然后把它们解析并保存在`Configuration`单例中。它会做如下初始化：
- `properties` 全局参数。
- `typeAliases` 别名。
- `Plugins` 插件。
- `objectFactory` 对象工厂。
- `objectWrapperFactory` 对象包装工厂。
- `reflectionFactory` 反射工厂。
- `settings` 环境设置。
- `environments` 数据库环境。
- `databaseldProvider` 数据库标识。
- `typeHandlers` 类型转换器。
- `Mappers` 映射器。

它们都会以类似`typeHandler`注册那样的方法被存放到`Configuration`单例中，以便未来将其取出。

#### 7.1.2 构建映射器的内部组成

当`XMLConfigBuilder`解析`XML`时，会将每一个`SQL`和其配置的内容保存起来。一般而言，在`MyBatis`中一条`SQL`和它相关的配置信息是由3个部分组成的，它们分别是`MappedStatement`、`SqlSource`和`BoundSql`
- `MappedStatement`的作用是保存一个映射器节点(`select|insert|delete|update`)的内容。它是一个类，包括许多我们配置的`SQL`、`SQL`的`id`、缓存信息、`resultMap`、`parameterType`、`resultType`、`resultMap`、`languageDriver`等重要配置内容。
- `SqlSource`是提供`BoundSql`对象的地方，它是`MappedStatement`的一个属性。它的作用是根据上下文和参数解析生成需要的`SQL`
- `BoundSql`是一个结果对象，也就是`SqlSource`通过解析得到的`SQL`和参数

![](src/200131_0.png)

对于最终的参数和SQL 都反映在`BoundSql`类对象上，在插件中往往需要拿到它进而可以拿到当前运行的`SQL`和参数，从而对运行过程做出必要的修改，来满足特殊的需求

`BoundSql`会提供3个主要的属性：`parameterMappings`、`parameterObject`和`sql`
1. `parameterObject`为参数本身，可以传递简单对象、`POJO`或者`Map`、`＠Param`注解的参数
2. `parameterMappings`是一个`List`，它的每一个元素都是`ParameterMapping`对象。对象会描述参数，参数包括属性名称、表达式、`javaType`、`jdbcType`、`typeHandler`等重要信息
3. `sql`属性就是书写在映射器里面的一条被`SqlSource`解析后的`SQL`

### 7.2 SqlSession运行过程

#### 7.2.1 映射器(Mapper)的动态代理

略(没看懂)

#### 7.2.2 SqlSession下的四大对象

`SqlSession`的执行过程是通过`Executor`、`StatementHandler`、`ParameterHandler`和`ResultSetHandler`来完成数据库操作和结果返回的
- `Executor`代表执行器，由它调度`StatementHandler`、`ParameterHandler`、`ResultSetHandler`
等来执行对应的`SQL`。其中`StatementHandler`是最重要的。
- `StatementHandler`的作用是使用数据库的`Statement`执行操作
- `ParameterHandler`是用来处理`SQL`参数的。
- `ResultSetHandler`是进行数据集`ResultSet`的封装返回处理的，它相当复杂，好在我们不常用它。

#### 7.2.3 SqlSession的运行

![](src/200131_1.png)

`SqlSession`是通过执行器`Executor`调度`StatementHandler`来运行的。而`StatementHandler`经过3步
- `prepared`预编译`SQL`
- `parameterize`设置参数
- `query/update`执行`SQL`

其中，`parameterize`是调用`parameterHandler`的方法设置的，而参数是根据类型处理器`typeHandler`处理的。`query/update`方法通过`ResultSetHandler`进行处理结果的封装，如果是`update`语句，就返回整数，否则就通过`typeHandler`处理结果类型，然后用`ObjectFactory`提供的规则组装对象，返回给调用者


