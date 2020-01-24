---
title: '[SSM][4][MyBatis配置]'
date: 2020-01-24 10:44:28
tags:
    - MyBatis
categories:
    - SSM
---
## 第4章 MyBatis配置

### 4.1 概述

MyBatis配置文件中所有元素清单如下
````xml
<? xml version="1.0" encoding="UTF-8"?>
<configuration>
    <properties/><!--属性-->
    <settings/><!--设置-->
    <typeAliases/><!--类型命名-->
    <typeHandlers><!--类型处理器-->
    <objectFactory/><!--对象工厂-->
    <plugins/><!--插件-->
    <environments><!--配置环境-->
        <environment><!--环境变量-->
            <transactionManager/><!--事务管理器-->
            <dataSource/><!--数据源-->
        </environment>
    <environments>
    <databaseIdProvider/><!--数据库厂商标识-->
    <mappers/><!--映射器-->
</configuration>
````
MyBatis配置项的顺序不能颠倒，否则在启动阶段会发生异常

### 4.2 properties属性

properties属性可以给系统配置一些运行参数，可以放在XML文件中，便于参数修改

MyBatis提供3种方式来使用properties
- `property`子元素
- `properties`文件
- 程序代码传递

#### 4.2.1 property子元素

````xml
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <properties>
        <property name="database.driver" value="com.mysql.cj.jdbc.Driver"/>
        <property name="database.url" value="jdbc:mysql://localhost:3306/mybatisdemo?serverTimezone=UTC"/>
        <property name="database.username" value="root"/>
        <property name="database.password" value="123456"/>
    </properties>
    <typeAliases>
        <typeAlias type="MyBatisDemo.pojo.Role" alias="role"/>
    </typeAliases>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"></transactionManager>
            <dataSource type="POOLED">
                <property name="driver" value="${database.driver}"/>
                <property name="url" value="${database.url}"/>
                <property name="username" value="${database.username}"/>
                <property name="password" value="${database.password}"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="MybatisDemo/mapper/RoleMapper.xml"/>
    </mappers>
</configuration>
````

这里使用了元素`<properties>`下的子元素`<property>`定义，用字符串`database.username`定义数据库用户名，然后就可以在数据库定义中引入这个已经定义好的属性参数，如`${database.username}`，这样可以定义一次到处引用

#### 4.2.2 使用properties文件

我们可以配置多个键值放在一个properties文件中，也可以把多个键值放到多个properties文件中。

先定义文件jdbc.properties
````
database.driver=com.mysql.cj.jdbc.Driver
database.url=jdbc:mysql://localhost:3306/mybatisdemo?serverTimezone=UTC
database.username=root
database.password=123456
````
然后在mybatis中通过`<properties>`的属性`resource`来引入这个文件
````xml
<properties resource="jdbc.properties"/>
````

#### 4.2.3 使用程序传递方式传递参数

略

#### 4.2.4 总结

这三种方式是具有优先级顺序的，MyBatis会根据优先级来覆盖原先配置的属性值。最优先的是使用程序传递的方式，其次是使用`properties`文件的方式，最后是使用`property`子元素的方式。

### 4.3 settings设置

settings可以深刻影响MyBatis底层的运行，但是在大部分情况下使用默认值就可以运行，所以不需要大量配置，只需要修改常用的规则即可

比如关于缓存的`cacheEnabled`，关于级联的`lazyLoadingEnabled`和`aggressiveLazyLoading`，关于自动映射的`autoMappingBehavior`和`mapUnderscoreToCamelCase`等等

### 4.4 typeAliases别名

由于类的全限定名很长，所以在大量使用时，一般采用一个简写来代表这个类，这就是别名

在MyBatis的初始化过程中，系统自动初始化了一些别名，例如`_int`、`_int[]`、`list`等，同时，`Configure`对象也对一些常用的配置类设置了别名，比如事务方式别名、数据源类型别名、缓存策略别名、语言驱动别名等。

MyBatis也提供了用户自定义别名的规则。我们可以通过`TypeAliasRegister`类的`registerAlias`方法注册，也可以采用配置文件或者扫描方式来自定义它。
````xml
<typeAliases>
    <typeAlias alias="role" type="com.neu.edu.pojo.Role"/>
    <typeAlias alias="user" type="com.neu.edu.pojo.User">
</typeAliases>
````

如果有很多类需要定义别名，可以通过扫描包的方式来实现
````xml
<typeAliases>
    <package name="com.neu.edu.pojo"/>
</typeAliases>
````

默认情况下，Mybatis会扫描这个包中的类，将其第一个字母变为小写作为其别名，例如`Role`的别名会变为`role`。若出现重名的情况，`Mybatis`允许使用注解`@Alias("role2")`的方式来进行避免
````
package com.neu.edu.pojo
@Alias("Role2")
public Class Role{
    ...
}
````

### 4.5 typeHandler类型转换器

`typeHandler`的作用是完成`jdbcType`和`javaType`之间的相互转换，其中`jdbcType`用于定义数据库类型，而`javaType`用来定义Java类型。

系统提供的`typeHandler`可以覆盖大部分场景的要求，假如有特殊需求，也可以自己定义特殊的转换规则

![](SSM-4-MyBatis配置/200124_0.png)

#### 4.5.1 系统定义的TypeHandler

在`MyBatis`中`typeHandler`都要实现接口`org.apache.ibatis.type.TypeHandler`，此接口的定义如下
````java
public interface TypeHandler<T>{
    void setParameter(PreparedStatement ps, int i, T parameter, jdbcType jdbcType) throws SQLException;
    T getResult(ResultSet rs, String columnName) throws SQLException;
    T getResult(ResultSet rs, int columnIndex) throws SQLException;
    T getResult(CallableStatement cs, int columnIndex) throws SQLException;
}
````
- 其中`T`是泛型，专指`javaType`，比如我们需要`String`的时候，实现类可以写为`implements TypeHandler<String>`
- `setParameter`方法，是使用`typeHandler`通过`PreparedStatement`对象进行设置SQL参数的时候使用的具体方法
- `getResult`方法，是从JDBC结果集中获取数据并进行转换，要么使用列名，要么使用下标

而`MyBatis`系统提供的`typeHandler`都继承了`org.apache.ibatis.type.BaseTypeHandler`

然后，采用`org.apache.ibatis.type.TypeHandlerRegister`类对象中的`register`方法进行注册

#### 4.5.2 自定义的TypeHandler

1. 首先要自定义一个类，要么实现`TypeHandler`接口，要么继承`BaseTypeHandler`类

2. 然后在配置文件中对这个新建的`typeHandler`进行注册
````xml
<typeHandlers>
    <typeHandler jdbcType="VARCHAR" javaType="string"
        handler="com.edu.neu.typeHandler.MyTypeHandler" />
</typeHandlers>
````

3. 配置完成后系统就会读取它，当`jdbcType`和`javaType`与`MyTypeHandler`对应的时候，就会启动`MyTypeHandler`

#### 4.5.3 枚举typeHandler

MyBatis对数据库的Blob字段也进行了支持，它提供了一个`BlobTypeHandler`

在现实中，一次性将大量数据加载到`JVM`中，会给服务器带来很大压力，所以在更多的时候要考虑使用文件流的形式，这时要将`POJO`的属性修改为`InputStream`


### 4.6 ObjectFactory(对象工厂)

当创建结果集时，`MyBatis`会使用一个对象工厂来完成创建这个结果集实例。在默认情况下，`MyBatis`会使用其定义的对象工厂`DefaultObjectFactory`来完成对应的工作

MyBatis也允许注册自定义的`ObjectFactory`
1. 实现`ObjectFactory`接口或者继承`DefaultObjectFactory`类
2. 在配置文件中注册
    ````xml
    <objectFactory type="com.edu.neu.MyObjectFactory">
        <property name="prop1" value="value1">
    </objectFactory>
    ````

### 4.7 插件

插件是`MyBatis`中最强大和灵活的组件，同时也是最复杂、最难使用的组件，它十分危险，会覆盖`MyBatis`底层对象的核心方法和属性

### 4.8 运行环境(Environment)

在`MyBatis`中，运行环境主要的作用是配置数据库信息，它可以配置多个数据库，一般而言只需要配置其中的一个就可以

它下面又分为两个可配置的元素：事务管理器(`transactionManager`)和数据源(`dataSource`)

#### 4.8.1 transactionManager(事务管理器)

transactionManager提供两个实现类，它需要实现接口`Transaction``
````java
public interface Transaction{
    Connection getConnection() throws SQLException;
    void commit() throws SQLException;
    void rollback() throws SQLException;
    void close() throws SQLException;
    Integer getTimeout() throws SQLException;
}
````
- 它的主要工作就是提交(commit)、回滚(rollback)和关闭(close)
- `MyBatis`为`Transaction`提供了两个实现类:`JdbcTransaction`和`ManagedTransaction`
- `JdbcTransaction`是以JDBC的方式对数据库的提交和回滚进行操作
- `ManagedTransaction`的提交和回滚不用任何操作，而是把事务交给容器处理

在配置文件中，可以通过如下配置，决定使用哪种`Transaction`方式
````xml
<transactionManager type="JDBC"/>
<transactionManager type="MANAGED"/>
````

#### 4.8.2 environment(数据源环境)

environment的主要作用是配置数据库
````xml
<environment>
    <dataSource type="UNPOOLED">
    <dataSource type="POOLED">
    <dataSource type="JNDI">
</environment>
````

##### 4.8.2.1 UNPOOLED

`UNPOOLED`采用非数据库池的管理方式，每次请求会打开一个新的数据库连接，所以创建会比较慢，在一些对性能没有很高要求的场合可以使用它

`UNPOOLED`类型的数据源可以配置如下几个属性
- `driver`数据库驱动名
- `url`连接数据库的URL
- `username`用户名
- `passowrd`密码
- `defaultTransactionIsolationLevel`默认的连接事务隔离级别

##### 4.8.2.2 `POOLED`

数据源`POOLED`利用池的概念将`JDBC`的`Connection`对象组织起来，它开始会有一些空置并且已经连接好的数据库连接，所以请求时，无须再建立和验证，省去去了创建新的连接实例时所必须的初始化和认证时间

##### 4.8.2.3 `JNDI`

数据源`JNDI`的实现是为了能在如`EJB`或应用服务器这类容器中使用

### 4.9 databaseIdProvider数据库厂商标识

此元素主要是支持多种不同厂商的数据库，来提升可移植性

### 4.10 引入映射器的方法

1. 用文件路径引入映射器
    ````xml
    <mappers>
        <mapper resourse="com/learn/ssm/mapper/roleMapper.xml"/>
    </mappers>
    ````

2. 用包名引入映射器
    ````xml
    <mappers>
        <package name="com.learn.ssm.mapper"/>
    </mappers>
    ````

3. 用类注册引入映射器
    ````xml
    <mappers>
        <mapper class="com.learn.ssm.mapper.UserMapper"/>
        <mapper class="com.learn.ssm.mapper.RoleMapper"/>
    </mappers>
    ````

4. 用`userMapper.xml`引入映射器
    ````xml
    <mappers>
        <mapper url="file:///var/mappers/com/learn/ssm/mapper/roleMapper.xml"/>
        <mapper url="file:///var/mappers/com/learn/ssm/mapper/userMapper.xml"/>
    </mappers>
    ````
