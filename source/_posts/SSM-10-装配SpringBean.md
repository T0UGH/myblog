---
title: '[SSM][10][装配SpringBean]'
date: 2019-12-25 13:01:39
tags:
    - Spring
categories:
    - SSM
---

## 10 装配SpringBean

### 10.1 依赖注入的3种方式

- Spring主要使用依赖注入的方式来完成IoC

- 依赖注入可以分为3种方式
  - 构造器注入
  - `setter`注入
  - 接口注入

#### 10.1 构造器注入

- 简介
  - 构造器注入依赖于构造方法的实现
  - 为了让`Spring`完成对应的构造注入，我们有必要去描述具体的类、构造方法并设置对应的参数，这样`Spring`就可以通过对应的信息用反射的形式创建对象

- 实例: 构造器注入
    1. 创建一个只有有参构造器的`POJO`
        ````java
        public class Role {
        private Long id;
        private String roleName;
        private String note;

        public Role(String roleName, String note) {
            this.roleName = roleName;
            this.note = note;
        }

        /*getter and setter*/
    }
        ````
    2. 在`spring-config.xml`中配置对象
        ````xml
        <bean id="role1" class="Role">
            <constructor-arg index="0" value="总经理"/>
            <constructor-arg index="1" value="公司管理者"/>
        </bean>
        ````

- 构造器配置
  - `constructor-arg`元素用于定义类构造方法的参数
  - `index`用于定义参数的位置
  - `value`则是设置值
  - 通过这样的定义Spring便知道使用`Role(String, String)`这样的构造方法去创建对象

#### 10.1.2 使用setter注入

- 简介
  - `setter`注入是Spring中最主流的注入方式，它利用`Java Bean`规范所定义的`setter`方法来完成注入，灵活且可读性高
  - 通过Java反射技术,Spring会调用没有参数的构造方法来生成对象，同时通过反射对应的`setter`注入配置的值

- 实例:配置`setter`注入
    ````xml
    <bean id="role2" class="Role">
        <property name="roleName" value="高级工程师"/>
        <property name="note" value="重要人员"/>
    </bean>
    ````

#### 10.1.3 接口注入

- 简介
  - 有些时候资源并非来自于自身系统，而是来自于外界
  - 比如数据库连接资源完全可以在Tomcat下配置，然后通过JNDI的方式去获取它

### 10.2 装配Bean概述

- 本节需要学习的是如何将自己开发的`Bean`装配到`Spring IoC`容器中

- Spring提供了3种方法进行配置
  - 在XML中显式配置
  - 在Java的接口和类中实现配置
  - 隐式Bean的发现机制和自动装配原则

- 3种方法的优先级
    1. 基于约定优于配置的原则，最优先的应该是通过隐式Bean的发现机制和自动装配的原则。这样的好处是减少程序开发者的决定权，简单而不失灵活
    2. 在没有办法使用自动装配原则的情况下应该优先考虑Java接口和类中实现配置，这样的好处是避免XML配置的泛滥
    3. 在上述方法都无法使用的情况下，只能选择XML去配置Spring IoC容器

### 10.3 通过XML配置装配Bean

- 首先给出XML文件的模板
    ````xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
        <!-- Spring Bean 配置代码 -->
    </beans>
    ````
    - 在上述代码中引入了一个`beans`的定义，它是一个根元素，而XSD文件也被引入了

#### 10.3.1 装配简易值

- 示例1: 简易XML装配`bean`
    ````xml
    <bean id="role2" class="Role">
        <property name="id" value="1"/>
        <property name="roleName" value="高级工程师"/>
        <property name="note" value="重要人员"/>
    </bean>
    ````
    - `id`属性是`Spring`对于这个`Bean`的编号，如果没有显式指定`id`，则`Spring`将采用`全限定名#{number}`的格式生成编号
    - `class`是一个类的全限定名
    - `property`元素是定义类的属性，其中`name`属性定义的是属性名称，而`value`是属性值

- 示例2: 有时需要向类A中注入自定义的类B
    ````xml
    <bean id="source" class="Source">
        <property name="fruit" value="橙汁"/>
        <property name="sugar" value="少糖"/>
        <property name="size" value="大杯"/>
    </bean>
    <bean id="juiceMaker" class="JuiceMaker">
        <property name="beverageShop" value="贡茶"/>
        <property name="source" ref="source"/>
    </bean>
    ````
    - 可以通过`ref`属性来引用对应的`Bean`

#### 10.3.2 装配集合

- Spring主要使用下列标签完成对集合的装配
    1. `List`标签为对应的`<list>`元素进行装配，然后通过多个`<value>`元素设值
    2. `Map`属性为对应的`<map>`元素进行装配，然后通过多个`<entry>`元素设值，只是`entry`包含一个键(key)和一个值(value)的设置。
    3. `Properties`属性，为对应的`<properties>`元素进行装配，通过多个`<property>`元素设置，只是`prop` 元素有一个必填属性`key`，然后可以设置值。
    4. `Set`属性为对应的`<set>`元素进行装配，然后通过多个`<value>`元素设值。
    5. 对于数组而言，可以使用`<array>`设置值，然后通过多个`<value>`元素设值。

- 示例: 装配集合类
    1. 首先定义一个复杂的类
        ````java
        public class ComplexAssembly {
        private Long id;
        private List<String> list;
        private Map<String, String> map;
        private Properties prop;
        private Set<String> set;
        private String[] array;
        }
        /*getter and setter*/
        ````
    2. 装配这个集合类
        ````xml
        <bean id="complexAssembly" class="ComplexAssembly">
            <property name="id" value="1"/>
            <property name="list">
                <list>
                    <value>value_list_1</value>
                    <value>value_list_2</value>
                    <value>value_list_3</value>
                </list>
            </property>
            <property name="map">
                <map>
                    <entry key="key1" value="value_key_1"/>
                    <entry key="key2" value="value_key_2"/>
                    <entry key="key3" value="value_key_3"/>
                </map>
            </property>
            <property name="prop">
                <props>
                    <prop key="prop1">value_prop_1</prop>
                    <prop key="prop2">value_prop_2</prop>
                    <prop key="prop3">value_prop_3</prop>
                </props>
            </property>
            <property name="set">
                <set>
                    <value>value_set_1</value>
                    <value>value_set_2</value>
                    <value>value_set_3</value>
                </set>
            </property>
            <property name="array">
                <array>
                    <value>value_array_1</value>
                    <value>value_array_2</value>
                    <value>value_array_3</value>
                </array>
            </property>
        </bean>
        ````

#### 10.3.3 命名空间装配

### 10.4 通过注解装配`Bean`

- 简介: 注解功能更加强大，它既可以实现`XML`的功能，也提供了自动装配的功能

- 在`Spring`中，它提供了两种方式来让`Spring IoC`容器发现`Bean`
    - 组件扫描: 通过定义资源的方式，让`Spring IoC`容器扫描对应的包，从而把`Bean`装配进来
    - 自动装配: 通过注解定义，使得一些依赖关系可以通过注解完成

- 通过扫描和自动装配，大部分的工程都可以用`Java`配置完成，而不是`XML`，这样可以有效减少配置和引入大量`XML`

#### 10.4.1 使用`@Component`装配`Bean`

- 示例1: 使用注解装配简单的`Bean`
    1. 定义`POJO`
        ````java
        package ComponentDemo;

        import org.springframework.beans.factory.annotation.Value;
        import org.springframework.stereotype.Component;

        @Component(value = "role")
        public class Role {
            @Value("1")
            private Long id;
            @Value("role_name_1")
            private String roleName;
            @Value("role_note_1")
            private String note;
        ````
        - 注解`@Component`代表`Spring IoC`会把这个类扫描生成`Bean`实例，其中的`value`属性代表这个类在`Spring`中的`id`
        - 注解`@Value`代表的是值的注入，这里只是注入了一些简单值
    2. 定义`Java Config`类
        ````java
        package ComponentDemo;

        import org.springframework.context.annotation.ComponentScan;

        @ComponentScan
        public class PojoConfig {
        }
        ````
        - 这个类的作用是告诉`Spring IoC`去哪里扫描对象
        - 注解`@ComponentScan`代表进行扫描，默认是扫描当前包的路径，`POJO`的包名和它保持一致时才能扫描
    3. 使用注解生成`Spring IoC`容器
        ````java
        package ComponentDemo;

        import org.springframework.context.ApplicationContext;
        import org.springframework.context.annotation.AnnotationConfigApplicationContext;

        public class Main {
            public static void main(String[] args) {
                ApplicationContext ctx = new AnnotationConfigApplicationContext(PojoConfig.class);
                Role role = ctx.getBean(Role.class);
                System.out.println(role.getNote());
                ((AnnotationConfigApplicationContext) ctx).close();
            }
        }
        ````

- `@ComponentScan`存在着两个配置项
    - `basePackages`: 它可以配置一个Java包的数组，`Spring`会根据它的配置扫描对应的包和子包，将配置好的`Bean`装配进来
    - `basePackageClasses`: 它可以配置多个类，`Spring`会根据配置的类所在的包，为包和子包进行扫描装配对应的配置`Bean`
    - 如果采用多个`@ComponentScan`去定义对应的包，但是每定义一个`@ComponentScan`，`Spring`就会为所定义的类产生一个新的对象
    - 对于`basePackages`和`basePackageClasses`来说，前一种可读性更好，所以会在项目中优先选择前一种

#### 10.4.2 自动装配---@Autowired

- 自动装配简介
    - `Spring`一般先完成`Bean`的定义和生成，然后再寻找需要注入的资源
    - 当`Spring`生成所有的`Bean`后，如果发现这个注解，它就会再`Bean`中查找，然后找到对应的类型，将其注入进来，这样就完成了依赖注入
    - 所谓自动装配技术是一种由`Spring`自己发现对应的`Bean`，自动完成装配工作的方式，它会应用到一个十分常见的注解`@Autowired`

- 示例: 一个简单的自动装配例子
    1. 定义`POJO`: 如前文的`Role`类
    2. 定义`RoleService`接口
        ````java
        package ComponentDemo;

        public interface RoleService {
            public void printRoleInfo();
        }

        ````
    3. 定义`RoleServiceImpl`实现类
        ````java
        package ComponentDemo;

        import org.springframework.stereotype.Component;

        @Component(value = "roleServiceImpl")
        public class RoleServiceImpl implements RoleService {
            @Autowired
            private Role role = null;
            @Override
            public void printRoleInfo() {
                System.out.println("id = " + role.getId());
                System.out.println("roleName = " + role.getRoleName());
                System.out.println("note = " + role.getNote());
            }
        }

        ````
    4. 定义`Java Config`类:略
    5. 测试:略

- 其他
    - `@Autowired`的注解表示在`Spring IoC`定位所有的`Bean`后，这个字段需要按类型注入，这样`IoC`容器会寻找资源，然后将其注入
    - IoC容器有时候会寻找失败，在默认的情况下寻找失败它就会抛出异常
      - 通过`@Autowired(required=false)`配置，可以在找不到资源时，不注入，但这样会产生空指针问题

#### 10.4.3 自动装配的歧义性(`@Primary`和`@Quaifier`)

- 背景: 在java中一个接口可能有多个实现类，这个时候`@Autowired`注解无法判断应该把哪个实现类的对象注入进来，从而抛出异常

- 方案1: 注解`@Primary`
    - 这个注解会告诉`Spring IoC`容器，优先使用拥有此注解的类注入
    - 例如
    ````java
    import org.springframework.context.annotation.Primary
    @Component("roleService3")
    @Primary
    public class RoleServiceImpl3 implements RoleService{}
    ````

- 方案2:注解`@Qualifier`
    - 这个注解会使用名称查找而不是类型查找的方式
    - 例如
    ````java
    public class RoleController{
        @Autowired
        @Qualifier("roleService3")
        private RoleService roleService = null;
        //...
    }
    ````

#### 10.4.4 装载带有参数的构造方法类

- 背景: 某些时候构造方法是有参数的，对于带参数的构造方法，我们也可以通过注解进行注入

- 方案: 使用`@Autowired`或者`@Qualifer`来修饰参数

- 例如
    ````java
    public RoleController(@Autowired RoleService roleService){
        this.roleService = roleService;
    }
    ````

#### 10.4.5 使用`@Bean`装配`Bean`

- 背景: 对于java来说，大部分开发都需要引入第三方的包，而且往往没有这些包的源码，这时候无法为这些包的类加入`@Component`注解，让它们成为开发环境中的`Bean`

- 方案:这个时候Spring给予一个注解`@Bean`，它可以注解到方法上，并且将方法返回的对象作为`Spring`的`Bean`，存放到`IoC`容器中

- 例如: 通过注解`@Bean`装配`DataSource`的`Bean`
    ````java
    @Bean(name="dataSource")
    public DataSource getDataSource(){
        Properties props = new Properties();
        props.setProperty("driver","com.mysql.jdbc.Driver");
        DataSource dataSource = null;
        try{
            dataSource = BasicDataSourceFactory.createDataSource(props);
        }catch(Exception e){
            e.printStackTrace();
        }
    }
    ````

- 这里还配置了`@Bean`的`name`选项为`dataSource`，这就意味着Spring生成该`Bean`的时候就会使用`dataSource`作为其`BeanName`。和其他`Bean`一样，它也可以通过`@Autowired`或者`@Qualifier`等注解注入到别的`Bean`中

### 10.5 装配的混合使用

Spring同时支持XML和注解两种装配方式，无论采用哪种，本质都是将`Bean`装配到`Spring IoC`容器中

建议
1. 在自己的工程中所开发的类尽量使用注解方法，因为使用它更为简单
2. 对于引入第三方或者服务的类，尽量使用`XML`方式，这样可以尽量减少对第三方包细节的理解，更清晰

如何将`xml`定义的`bean`引入`java`配置中?
- 可以使用注解`@ImportResourse`，它可以配置多个`XML`配置文件，将这些文件中定义的`bean`全部引入
- 例如
    ````java
    package com.ssm.annotation.config
    import org.springframework.context.anotation.ComponentScan;
    import org.springframework.context.anotation.ImportResource;
    @ComponentScan(basePackages={"com.ssm.chapter10.annotation"})
    @ImportResource({"classpath:spring-dataSource.xml"})
    public class ApplicationConfig{}
    ````

### 10.6 使用Profile

背景: 在软件开发中，可能开发人员使用一套环境，而测试人员使用另一套环境，这就有了在不同的环境中进行切换的需求。Spring也对这样的场景进行了支持，在Spring中可以定义`Bean`的`Profile`，方式有两种
1. 使用注解`@Profile`配置
2. 使用`xml`定义`Profile`

#### 10.6.1 使用注解@Profile配置

例子如下
````java
package com.ssm.chapter10.profile;
@Component
public class ProfileDataSource{
    @Bean(name="devDataSource")
    @Profile("dev")
    public DataSource getDevDataSource() {
        Properties props= new Properties();
        props.setProperty("driver", "com.mysql.jdbc.Driver");
        props.setProperty("url", "jdbc:mysql://localhost:3306/chapter12");
        props.setProperty("username", "root");
        props.setProperty("password", "123456");
        DataSource dataSource = null;
        try{
            dataSource = BasicDataSourceFactory.createDataSource(props);
        }catch(Exception e){
            e.printStackTrace();
            return dataSource;
        }
    }
    @Bean(name="testDataSource")
    @Profile("test")
    public DataSource getTestDataSource(){
        Properties props=new Properties();
        props.setProperty("driver", "com.mysql.jdbc.Driver");
        props.setProperty("url", "jdbc:mysql://localhost:3306/chapter13");
        props.setProperty("username", "root");
        props.setProperty("password", "123456");
        DataSource dataSource = null;
        try{
            dataSource = BasicDataSourceFactory.createDataSource(props);
        }catch(Exception e){
            e.printStackTrace();
            return dataSource;
        }
    }
}
````

#### 10.6.2 使用xml定义Profile

例子如下
````xml
<?xml version='1.0' encoding='UTF-8'?>
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p" xsi:schemaLocation="http://www.springframework.org/schema/beans">
    <beans profile="test">
        <bean id="devDataSource" class="org.apache.commons.dbcp.BaseDataSource">
            <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
            <property name="url" value="jdbc:mysql://localhost:3306/chapter12"/>
            <property name="username" value="root"/>
            <property name="password" value="123456"/>
        </bean>
    </beans>
        <beans profile="dev">
        <bean id="devDataSource" class="org.apache.commons.dbcp.BaseDataSource">
            <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
            <property name="url" value="jdbc:mysql://localhost:3306/chapter13"/>
            <property name="username" value="root"/>
            <property name="password" value="123456"/>
        </bean>
    </beans>
````

#### 10.6.3 启动Profile

当启动`Java`配置或者`XML`配置`Profile`时，可以发现这两个`Bean`并不会被加载到`SpringIoC`容器中，需要自行激活`Profile`。激活`Profile`的方法有5种
1. 在使用`SpringMVC`的情况下可以配置`web`上下文参数，或者`DispatchServlet`参数
2. 作为`JNDI`条目
3. 配置环境变量
4. 配置`JVM`启动参数。
5. 在集成测试环境中使用`＠ActiveProfiles`

### 10.7 加载属性(properties)文件

略

### 10.8 条件化装配Bean

背景: 在某些条件下需要进行条件判断，判断是否需要装配`Bean`。

方案: 这时`Spring`提供了注解`@Conditional`去配置，通过它可以配置一个或多个实现了`Condition`接口的类

例子:
1. 如何放置条件判断
    ````java
    @Bean(name="dataSource")
    @Condition({DataSourceCondition.class})
    public DataSource getDataSource(){
        //.....
    }
    ````
2. 条件判断类的实现
    ````java
    public class DataSourceCondition implements Condition{
        @Override
        public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata){
            //....
        }
    }
    ````
    - 这里要求实现接口`Condition`的`matches`方法。如果此方法返回`true`则`Spring`会创建对应的`Bean`

### 10.9 Bean的作用域

在默认情况下,`SpringIoc`容器只会对一个`Bean`创建一个实例，有时候我们希望容器可以产生多实例，就需要修改`Spring`的作用域

`Spring`提供4种作用域
- 单例(singleton): 默认选项，在整个应用中，`Spring`只会为其生成一个`Bean`实例
- 原型(prototype): 当每次通过容器获取`Bean`时都会创建一个实例
- 会话(session): 在会话过程中`spring`只创建一个实例
- 请求(request): 在一个请求中`spring`只创建一个实例

### 10.10 使用Spring表达式(SpringEL)

SpringEL的主要功能
- 使用`Bean`的`id`来引用`Bean`
- 使用指定对象的方法和访问对象的属性
- 进行运算
- 提供正则表达式进行匹配
- 集合配置

也就是为注解提供一定的逻辑运算功能，具体使用方式略





