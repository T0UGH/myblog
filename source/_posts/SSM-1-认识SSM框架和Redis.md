---
title: '[SSM][1][认识SSM框架和Redis]'
date: 2019-06-19 00:37:54
tags:
    - Spring
    - Redis
    - SpringMVC
    - MyBatis
categories:
    - SSM
---
## 第 1 章 认识SSM框架和Redis

### 1.1 Spring框架

- Spring框架是Java应用最广的框架，它的成功来源于理念而不是技术本身

- 它的理念包括IoC(Inversion of Control, 反转控制)和AOP(Aspect Oriented Programming, 面向切面编程)

#### 1.1.1 Spring Ioc 简介

- IoC定义: IoC是一个容器，在Spring中，它认为所有Java资源都是JavaBean，容器的目标就是管理这些Bean和它们之间的关系

- 控制反转: Spring是依靠描述来完成对象的创建及其依赖关系的。这是一种被动的行为，需要的资源通过描述信息就可以得到，其中的控制权在Spring IoC容器中，它会根据描述找到使用者需要的资源，这就是控制反转的含义

- 好处
  - 这样做的好处是Socket接口不再依赖于某个实现类，需要使用某个实现类时我们通过配置信息就可以完成了。
  - 这样想修改或者加入其他资源就可以通过配置信息完成，不需要再用`new`关键字创建对象，依赖关系也可以通过配置完成，从而完全可以即插即用地管理它们之间的关系
  
- Spring IoC的理念
  - 你不需要去找资源，只要向IoC容器描述所需资源，IoC自己会找到你所需要的资源，这就是Spring IoC的理念
  - 这样就把Bean之间的依赖关系解耦了，更容易写出结构清晰的程序

#### 1.1.2 Spring AOP

- 只用OOP并不完善，还需要面向切面的编程，通过它去管理在切面上的某些对象的协作
- AOP(面向切面编程)常用于数据库事务的编程
  - 在默认情况下，只要Spring接收到了异常消息，它就会将数据路的事务回滚，从而保障数据的一致性
  - 一些复杂的数据库代码、try..catch等被Spring隐藏了，只需要关注业务代码，知道只要发生了异常情况，Spring就会回滚事务即可

- AOP举例
````java
/**
* Spring AOP 处理订单伪代码
* @param order 订单
*/
private void proceed(Order order){
    boolean pflag = productionDept.isPass(order);
    if(pflag){
        if(financialDept.isOverBudget(order)){
            throw new RuntimeException("预算超额!!");
        }
    }
}
````

### 1.2 MyBatis简介

- MyBatis的优势在于灵活，它几乎可以代替JDBC，同时提供了接口编程。
  
- 目前MyBatis的数据访问层DAO(Data Access Objects)是不需要实现类的，它只需要一个接口和XML

- MyBatis提供自动映射、动态SQL、级联、缓存、注解、代码和SQL分离等特性，使用方便，同时也可以对SQL进行优化

- Pojo: Plain Ordinary Java Object

- 无论是MyBatis还是Hibernate都是依靠某种方式，将数据库的表和POJO映射起来的，这样程序员就可以操作POJO来完成相关的逻辑了
  
#### 1.2.1 Hibernate简介

- 要将POJO和数据库映射起来需要给这些框架提供映射规则

- 在MyBatis或Hibernate中可以通过XML或者通过注解来提供映射规则

- ORM: 我们把POJO对象和数据库表相互映射的框架称为对象关系映射(Object Relational Mapping, ORM)框架，无论Mybatis还是Hibernate都是ORM框架

- Hibernate基本不再需要编写SQL就可以通过映射关系来操作数据库，是一种全表映射的体现;MyBatis不同，它需要我们提供SQL去运行

- Hibernate是将POJO和数据库表对应的映射文件，如下所示
    ````xml
    <?xml version="1.0"?>
    <! DOCTYPE hibernate-mapping PUBLIC "-//Hibernate/Hibernate Mapping DTD 3.0//EN" "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">
    <hibernate-mapping>
        <class name="com.learn.chapter1.pojo.Role" table="t_role">
            <id name="id" type="java.lang.Integer">
                <column name="id" />
                <generator class="identity" />
            </id>
            <property name="roleName" type="string">
                <column name="role_name" length="60" not-null="true" />
            </property>
            <property name="note" type="string">
                <column name="note" length="512" />
            </property>
        </class>
    </hibernate-mapping>
    ````
    - 首先对POJO和表t_role进行了映射配置，把两者映射起来。
    - 然后，对POJO进行操作，从而影响t_role表的数据

- 如何使用Hibernate操作数据库数据
    ````java
    Session session = null;
    Transaction tx = null;
    try{
        //打开会话
        session = HibernateUtil.getSessionFactory().openSession();
        //打开事务
        tx = session.beginTransaction();
        Role role = new Role();
        role.setId(1);
        role.setRoleName("rolename1");
        role.setNote("note1");
        session.save(role);//保存
        Role role2 = (Role)session.get(Role.class, 1);
        role2.setNote("修改备注");
        session.update(role2);
        System.out.println(role2.getRoleName());
        session.delete(role2);
        tx.commit();
    }catch(Exception ex){
        if(tx != null && tx.isActive()){
            tx.rollback();
        }
        ex.printStackTrace();
    }finally{
        if(session != null && session.isOpen()){
            session.close();
        }
    }
    ````

#### 1.2.2 MyBatis

- 简介
  - Mybatis不屏蔽SQL
  - 不屏蔽SQL的优势在于，程序员可以自己指定SQL规则，无需Hibernate自动生成规则，这样可以更加精确地定义SQL，从而优化性能。
  - 它更符合移动互联网高并发、大数据、高性能、高响应的要求

- MyBatis需要一个映射文件把POJO和数据库的表对应起来
    ````xml
    <?xml version="1.0" encoding="utf-8" ?>
    <! DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
    <mapper>
        <resultMap id="roleMap" type="com.learn.chapter1.pojo.Role">
            <id property="id" column="id" />
            <result property="roleName" column="role_name" />
            <result property="note" column="note" />
        </resultMap>

        <select id="getRole" resultMap="roleMap">
            select id, role_name, note from t_role where id = #{id}
        </select>

        <delete id="deleteRole" parameterType="int">
            select form t_role where id = #{id}
        </delete>

        <insert id="insertRole" parameterType="com.learn.chapter1.pojo.Role">
            update t_role set
            role_name = #{roleName}
            note = #{note}
            where id = #{id}
        </insert>
    </mapper>

    ````
    - 这里的resultMap元素用于定义映射规则
    - 增删改查对应着insert、delete、select、update四大元素，十分明了

- mapper元素中的namespace属性，它要和一个接口的全限定名一致，而里面的SQL的id也需要和接口定义的方法完全一致。Mybatis接口文件如下
  ````java
  package com.learn.chapter1.mapper;
  import com.learn.chapter1.pojo.Role;
  public interface RoleMapper{
 
    public Role getRole(Integer id);
 
    public int deleteRole(Integer id);

    public int insertRole(Role role);

    public int updateRole(Role role);
  }
  ````
  - 定义完这个MyBatis映射文件后，并不需要定义一个实现类

- MyBatis如何操作数据
  ````java
  SqlSession sqlSession = null;
  try{
      sqlSession = MyBatisUtil.getSqlSession();
      RoleMapper roleMapper = sqlSession.getMapper(RoleMapper.class);
      Role role = roleMapper.getRole(1);//查询
      System.out.printlen(role.getRoleName());
      role.setRoleName("update_role_name");
      roleMapper.updateRole(role);
      Role role2 = new Role();
      role2.setNote("note2");
      role2.setRoleName("role2");
      roleMapper.insertRole(role);
      roleMapper.deleteRole(5);
      sqlSession.commit();
  }catch(Exception ex){
      ex.printStackTrace();
      if(sqlSession != null){
          sqlSession.rollback();
      }
  }finally{
      if(sqlSession != null){
          sqlSession.close();
      }
  }
  ````

#### 1.2.3 比较Hibernate和MyBatis

- Hibernate优缺点
  - 优点
    1. 不需要编写大量的SQL，就可以完成映射
    2. 同时提供了日志、缓存、级联等特性
    3. 此外提供了HQL(Hibernate Query Language)对POJO进行操作，十分方便
  - 缺点
    1. 由于无须SQL，当多表关联超过3个的时候，通过Hibernate的级联会造成太多性能的丢失
    2. 遇到存储过程，Hibernate只能就此作罢
    3. 不适用性能要求苛刻的系统

- MyBatis优缺点
  - 优点
    1. MyBatis可以自由书写SQL、支持动态SQL、处理列表、动态生成表名、支持存储过程
    2. 可以灵活的定义查询语句，满足各类需求和性能优化的需要
  - 缺点
    1. 要编写SQL和映射规则，其工作量大于Hibernate
    2. 它支持的工具也很有限，不能像Hibernate那样有许多的插件可以帮助生成映射代码和关联关系

- 使用场景
  - 对于性能要求不太严苛的系统，比如管理系统、ERP等推荐使用Hibernate
  - 对于性能要求高、响应快、灵活的系统推荐使用MyBatis

### 1.3 SpringMVC 简介

- 长期以来Struts2与Spring的结合一直存在很多问题，比如兼容性和类臃肿

- SpringMVC结构层次清晰，类比较简单，并且与Spring的核心IoC和AOP无缝衔接

- MVC模式把应用程序分成不同的方面，同时提供了这些元素之间的松耦合
  - Model(模型): 封装了应用程序的数据和由它们组成的POJO
  - View(视图): 负责吧模型数据渲染到视图上，将数据以一定的形式展现给用户
  - Controller(控制器): 负责处理用户请求，并建立适当的模型把它传递给视图渲染

- SpringMVC的重点在于它的流程和一些重要的注解，包括控制器、视图解析器、视图等重要内容

### 1.4 NoSQL与Redis

- NoSQL(Not Only SQL)
    - 它具备一定持久层的功能
      - 它存储的数据是半结构化的，这就意味着计算机在读入内存中有更少的规则，读入速度更快
    - 也可以作为一种缓存工具
      - 它可以支持大数据存入内存中，只要命中率高，它就能快速响应

- NoSQL的作用
  - 对于常见的数据，第一次从数据库读出，然后就存放到NoSQL中，这样以后就无须再访问数据库，只需从NoSQL中读出即可，可提高性能
  - 对于一些高并发的操作，可以再NoSQL上先完成写入，等待某个时刻再批量写入数据库，这样就能满足系统的性能要求

- 传统数据库的好处
  - 更好的规范性和数据完整性
  - 功能更强大，作为持久层更完善，安全性更高

- Redis作为主要的NoSQL工具的优点
    - 响应快速
    - 支持六种数据类型:字符串、哈希结构、列表、集合、可排序集合和基数
    - 操作都是原子的
    - MultiUtility工具: Redis可以在如缓存、消息队列中使用，在应用程序如Web程序会话、网站页面点击数等任何短暂的数据中使用

- NoSQL的优势
  - 一方面，使用NoSQL从数据库中读取数据进入缓存，就可以从内存中读取数据，而不像数据库从磁盘读数据。现实是读操作远比写操作多得多，所以缓存很多常用的数据，提高其命中率有助于整体性能的提高，并且缓解数据库的压力，对互联网系统架构十分有用
  - 另一方面，它能满足互联网高并发需要高速处理数据的场合，比如抢红包、商品秒杀等场景，这些场景需要高速处理，并保证并发数据安全和一致性

### 1.5 SSM+Redis结构框图及概述

- 结构框图如下

- 各自承担的功能
  - Spring IoC承担一个资源管理、整合、即插即用的功能
  - Spring AOP可以提供切面管理，特别是数据库事务管理的功能
  - SpringMVC可以把模型、视图和控制器分层，组合成一个有机灵活的系统
  - MyBatis提供了一个数据库访问的持久层，通过MyBatis-Spring项目，它便能和Spring无缝对接
  - Redis作为缓存工具，提供高速处理数据和缓存数据的功能，使得系统大部分是需要访问缓存，而无需从数据库磁盘中重复读/写