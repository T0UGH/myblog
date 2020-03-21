---
title: '[MyBatis源码][2][配置文件的解析过程]'
date: 2020-03-21 14:14:37
tags:
    - MyBatis
categories:
    - MyBatis
---
## 第二章 配置文件的解析过程

首先我们从一个入口的例程开始：

````java
//id:2.0
public class MyApp {
    public static void main(String[] args) throws IOException {
        Logger logger = Logger.getLogger(MyApp.class);
        InputStream inputStream = Resources.getResourceAsStream("mybatis.xml");
        SqlSessionFactory sqlSessionFactory = 
            new SqlSessionFactoryBuilder().build(inputStream);
        SqlSession sqlSession = sqlSessionFactory.openSession();
        ProductDao productDao = sqlSession.getMapper(ProductDao.class);
        Product product= productDao.getProduct(12);
        logger.info(product);
    }
}
````

从这个例程我们可以看出：

1. 首先我们通过MyBatis提供的`Resources`类读取了配置文件。
2. 然后使用`SqlSessionFactoryBuilder`，来建造一个`SqlSessionFactory`
3. 接着通过这个工厂获取`SqlSession`实例，就可以使用`SqlSession`执行各种数据库操作



接下来详细分析`SqlSessionFactoryBuilder`的`build()`方法

```java
//id:2.1
// package org.apache.ibatis.session;
// SqlSessionFactoryBuilder
public SqlSessionFactory build(InputStream inputStream, 
                                 String environment, 
                                 Properties properties) {
    try {
      XMLConfigBuilder parser = 
          new XMLConfigBuilder(inputStream, environment, properties);
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        inputStream.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }

  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
```

可以看到，主要逻辑是，使用`XMLConfigBulider`的`parse()`方法生成一个`Configuration`对象，`Configuration`类存放了Mybatis的所有全局配置，根据这个类中的配置，我们就可以使用建造者模式建造一个`SqlSessionFactory`了。



我们再跟随第九行代码的调用栈向下

```java
//id:2.2
//package org.apache.ibatis.builder.xml;  
//XMLConfigBuilder
public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }
```

这个方法的逻辑极其简单，首先通过一个标识`parsed`判断这个配置文件是否已经被解析过了，解析过则直接返回。否则调用`parseConfiguration()`进行解析。



我们再跟随第8行代码的调用栈向下

```java
//id:2.3
//package org.apache.ibatis.builder.xml;  
//XMLConfigBuilder 
private void parseConfiguration(XNode root) {
    try {
      	//issue #117 read properties first
     	propertiesElement(root.evalNode("properties"));
        
      	Properties settings = settingsAsProperties(root.evalNode("settings"));
      
      	loadCustomVfs(settings);
      	loadCustomLogImpl(settings);
      
      	typeAliasesElement(root.evalNode("typeAliases"));
      
      	pluginElement(root.evalNode("plugins"));
      
        objectFactoryElement(root.evalNode("objectFactory"));
      
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
      
        settingsElement(settings);
      
        // read it after objectFactory and objectWrapperFactory issue #631
      
        environmentsElement(root.evalNode("environments"));
      
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      
        typeHandlerElement(root.evalNode("typeHandlers"));
      
        mapperElement(root.evalNode("mappers"));
        
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " 
                                 + e, e);
    }
}
```

可以看到这个方法按照顺序，从`XNode`根节点出发，解析了不同的配置文件节点。这里简单解释一下什么是`XNode`，它是一颗类似DOM结构的树，这棵树存放了XML的初步解析结果。



上述调用栈如下图

![](https://i.loli.net/2020/03/21/5HmuiwlYjrStg4a.png)

接下来我们将具体分析`<properties>`节点、`<settings>`节点、`<typeAliases>`节点和`<typeHandler>`节点的解析过程，注意：这些节点的解析结果最终都将存放在`Configuration`中



### 2.1 `<properties>`节点解析过程

`<properties>`节点的主要作用是定义一些在后面的节点会使用的变量，如下

```xml
<!--id:2.4-->
<configuration>
    <properties>
        <property name="jdbc.driver" value="com.mysql.cj.jdbc.Driver"/>
        <property name="jdbc.url" value="jdbc:mysql://localhost:3306/cosmetic_store?serverTimezone=UTC"/>
        <property name="jdbc.username" value="root"/>
        <property name="jdbc.password" value="123456"/>
    </properties>
    <settings>
        <setting name="cacheEnabled" value="true"/>
    </settings>
    <typeAliases>
        <typeAlias type="biz.t0ugh.Model.Product" alias="product"/>
    </typeAliases>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="ProductMapper.xml"/>
    </mappers>
</configuration>
```

可以看到，我们在`<dataSource>`中引用了`<properties>`节点定义的变量



接下来我们从编号`2.3`的代码段第7行`propertiesElement(root.evalNode("properties"));`向下，详细分析这个方法。



首先我们知道，`properties`的内容由两部分构成

1. 可以在子节点`<property>`中，通过键值对的方式定义，如`2.4`代码段中从4-7行所示
2. 也可以通过`<properties>`节点上的`resource`属性或者`url`属性，从其他文件中读入一些`properties`



那么这个方法实际上也只是做了这项工作

1. 遍历所有`<property>`子节点，将得到的k-v对存入一个`Properties`的实例
2. 从其他文件中读取，将得到的所有k-v对存入第一步的`Properties`实例。注意：若第二步中某些配置与第一步重名，第一步的这个配置将被覆盖
3. 最后将得到的`Properties`实例存入到`Configuration`中，这样以后MyBatis的其他部分使用这些属性时直接读取即可

源代码如下

```java
//id:2.5
//package org.apache.ibatis.builder.xml;  
//XMLConfigBuilder 
private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
        
      	//1. 首先解析所有子节点并放入defaults
      	Properties defaults = context.getChildrenAsProperties();
      
        //2. 读取文件中的property，也放入defaults
       	String resource = context.getStringAttribute("resource");
      	String url = context.getStringAttribute("url");
      	if (resource != null && url != null) {
        	throw new BuilderException("xxxx");
      	}
      	if (resource != null) {
        	defaults.putAll(Resources.getResourceAsProperties(resource));
      	} else if (url != null) {
        	defaults.putAll(Resources.getUrlAsProperties(url));
      	}
        
        //3.读取Configuration中原本的property，放入defaults
      	Properties vars = configuration.getVariables();
      	if (vars != null) {
        	defaults.putAll(vars);
      	}
        
        //4.将default存入Configuration
      	parser.setVariables(defaults);
      	configuration.setVariables(defaults);
    }
  }
```



### 2.2 `<settings>`节点解析过程



`<settings>`节点的主要作用是定义一些MyBatis运行时的行为，如代码块`2.4`的9-11行中，定义了是否开启缓存



接下来我们从编号`2.3`的代码段第9行`Properties settings = settingsAsProperties(root.evalNode("settings"));`向下，详细分析这个方法。



首先我们思考`settings`解析与`properties`解析的区别是什么？答案是`<setting>`节点中的`name`属性必须是mybatis支持的配置，要言之有物才行。假如我们如下代码定义一个名为`hello`的节点，mybatis一定会报错，因为它没有`hello`这个设置。

```xml
<settings>
    <setting name="cacheEnabled" value="true"/>
    <!--下面这个设置是不存在的，肯定会报错-->
    <setting name="hello" value="world"/>
</settings>
```



那么我们接着思考，如何验证一个设置是否存在呢？最简单的方法是维护一个常量表，它存储所有存在的设置。但是这很不灵活还会造成冗余。Mybatis使用了Java的放射机制，它有一个工具类叫做`MataClass`，可以读取`Configuration`中的所有`setxxx()`方法。这样我们每读取一个设置，就通过`MataClass`来检查`Configuration`中是否有对应的`setter`，没有就报错。



具体步骤如下

1. 解析`settings`子节点的内容，并将解析结果转成`Properties`对象
2. 为`Configuration`创建元信息对象`MetaClass`
3. 通过`MetaClass`检测`Configuration`中是否存在某个属性的`setter`方法
4. 若通过`MetaClass`的检测，则返回`Properties`对象。



```java
//id:2.5
//package org.apache.ibatis.builder.xml;  
//XMLConfigBuilder 
private Properties settingsAsProperties(XNode context) {
    if (context == null) {
    	return new Properties();
    }
    //1. 解析settings子节点的内容，并将解析结果转成Properties对象
    Properties props = context.getChildrenAsProperties();
    
    //2.生成元数据对象
    // Check that all settings are known to the configuration class
    MetaClass metaConfig = 
        MetaClass.forClass(Configuration.class, localReflectorFactory);
    
    //3.对每个设置项进行检查
    for (Object key : props.keySet()) {
      	if (!metaConfig.hasSetter(String.valueOf(key))) {
      		throw new BuilderException("The setting " + key + " is not known.");
      	}
    }
    
    //4.返回
    return props;
}
```



这样我们就取得了包含所有设置项的`Properties`对象，接下来还需要将这个对象中的内容存储到`Configuration`中，源代码如下，逻辑很简单，就是调用各种`setter`而已。

```java
//id:2.6
//package org.apache.ibatis.builder.xml;  
//XMLConfigBuilder 
private void settingsElement(Properties props) {
   		configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
    //...其他省略，都是这种setter
}
```



### 2.3 `<typeAliases>`节点解析过程



我们都知道，MyBatis提供一个很方便的功能：我们可以给自己写的类定义别名(aliases)，这样当我们在MyBatis中需要使用类名时，只需要写这个别名，而不需要写冗长的全限定类名。



MyBatis中的别名配置方式有两种

1. 配置包名，这个包下的所有类都会被扫描并且根据类名生成别名

   ```xml
   <typeAliases>
   	<package name="com.edu.neu.zady.dao"/>
   </typeAliases>
   ```

2. 通过手动的方式，明确为某个类型配置别名

   ```xml
   <typeAliases>
   	<package alias="product" type="com.edu.neu.zady.dao.Product"/>
   </typeAliases>
   ```



除了这些自定义的别名，MyBatis还在`Configuration`中为一些常用类生成了别名。



在`Configuration`中，自定义的别名和预定义的别名都存放在了`TypeAliasRegister`中，它提供注册别名和获取别名的功能。



具体源代码如下

```java
//id:2.7
//package org.apache.ibatis.builder.xml;  
//XMLConfigBuilder 
private void typeAliasesElement(XNode parent) {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        // 1. 解析使用package配置的别名-类型映射
        if ("package".equals(child.getName())) {
          String typeAliasPackage = child.getStringAttribute("name");
          configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
        } else {// 2. 解析使用typeAlias配置的别名-类型映射
          String alias = child.getStringAttribute("alias");
          String type = child.getStringAttribute("type");
          try {
            Class<?> clazz = Resources.classForName(type);
            if (alias == null) {
              typeAliasRegistry.registerAlias(clazz);
            } else {
              typeAliasRegistry.registerAlias(alias, clazz);
            }
          } catch (ClassNotFoundException e) {
            throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
          }
        }
      }
    }
  }
```



由上源码可见，对于别名的解析，主要是使用`Configuration`中的`typeAliasRegister`属性的各种重载的`registerAlias()`方法，这些重载比较多。我们挑选代码段`2.7`的第19行`typeAliasRegistry.registerAlias(alias, clazz);`作为入口，继续向下，分析一个具体的注册过程

```java
//id:2.8
//package org.apache.ibatis.type;
//TypeAliasRegistry

//成员变量，一个Map，用来实际存放别名-类型映射
private final Map<String, Class<?>> typeAliases = new HashMap<>();

public void registerAlias(String alias, Class<?> value) {
    
    if (alias == null) {
      throw new TypeException("The parameter alias cannot be null");
    }
    //1. 首先将名称alias转化为小写
    // issue #748
    String key = alias.toLowerCase(Locale.ENGLISH);
    
    //2. 判断这个alias是否已经注册过了，注册过了就报错
    if (typeAliases.containsKey(key) 
        && typeAliases.get(key) != null 
        && !typeAliases.get(key).equals(value)) {
        
      throw new TypeException("The alias '" + 
                              alias + 
                              "' is already mapped to the value '" +
                              typeAliases.get(key).getName() +
                              "'.");
    }
    
    //3. 存入Map中
    typeAliases.put(key, value);
  }
```



解析`<package>`的过程与此类似，这里就不再赘述。



### 2.4 `<typeHandler>`节点解析过程



在向数据库存储或读取数据时，我们需要将数据库字段类型和java类型进行一个转换。比如数据库中有`CHAR`和 `VARCHAR`等类型，但java中没有这些类型，不过java有`String`类型。所以我们在从数据库中读取`CHAR`和 `VARCHAR`类型的数据时，就可以把它们转成`String`。在 MyBatis中，数据库类型和java类型之间的转换任务是委托给类型处理器`TypeHandler`去处理的。`MyBatis`提供了一些常见类型的类型处理器，除此之外，我们还可以自定义类型处理器以非常见类型转换的需求。





了解完`TypeHandler`的用途，我们继续探究它是如何注册到`Configuration`的。我们从代码段`2.3`的32行`typeHandlerElement(root.evalNode("typeHandlers"));`继续向下，查看`typeHandlerElement`方法，这个方法主要有三步

1. 读取javaType、jdbcType、handlerType的字符串形式，也就是类名
2. 将这些类名解析为具体的Class对象
3. 根据前两步的解析结果选择不同的解析方法，也是一堆register的重载方法

```java
//id:2.9
//package org.apache.ibatis.builder.xml;  
//XMLConfigBuilder 
private void typeHandlerElement(XNode parent) {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        //使用指定的包来注册TypeHandler
        if ("package".equals(child.getName())) {
          String typeHandlerPackage = child.getStringAttribute("name");
          typeHandlerRegistry.register(typeHandlerPackage);
        } else {//使用typeHandlers的typeHandler子节点来注册TypeHandler
          
          //1. 读取javaType、jdbcType、handlerType的字符串形式，也就是类名
          String javaTypeName = child.getStringAttribute("javaType");
          String jdbcTypeName = child.getStringAttribute("jdbcType");
          String handlerTypeName = child.getStringAttribute("handler");
            
          //2. 将这些类名解析为具体的Class对象
          Class<?> javaTypeClass = resolveClass(javaTypeName);
          JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
          Class<?> typeHandlerClass = resolveClass(handlerTypeName);
          
          //3. 根据前两步的解析结果选择不同的解析方法，也是一堆register的重载方法
          if (javaTypeClass != null) {
            if (jdbcType == null) {
              typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
            } else {
              typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
            }
          } else {
            typeHandlerRegistry.register(typeHandlerClass);
          }
        }
      }
    }
 }
```



我们现在知道了，实际的解析过程是通过`Configuration`的成员变量`TypeHandlerRegistry`的各种名为`register`的重载方法进行的。我们从代码段`2.9`的28行继续向下`typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);`，这个方法的具体分析我放到了注释中。



```java
//id:2.10
//package org.apache.ibatis.type;
//TypeHandlerRegister

//这个Map<Type, Map>实际保存了JavaType到JdbcType的映射，并且还保存了JdbcType到TypeHandler的映射
private final Map<Type, Map<JdbcType, TypeHandler<?>>> typeHandlerMap 
    = new ConcurrentHashMap<>();

//register的一个重载方法
private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
    if (javaType != null) {
      //1. 在typeHandlerMap(映射保存实际使用的数据结构)中，使用javaType作为key查询，返回一个保存了这个javaType所有映射的Map
      Map<JdbcType, TypeHandler<?>> map = typeHandlerMap.get(javaType);
      
      //2.如果为空，则创建
      if (map == null || map == NULL_TYPE_HANDLER_MAP) {
        map = new HashMap<>();
        typeHandlerMap.put(javaType, map);
      }
      
      //3.将通过参数传入的这个新的类型处理器加入map
      map.put(jdbcType, handler);
    }
    allTypeHandlersMap.put(handler.getClass(), handler);
}
```



由代码段`2.10`可知，注册过程其实就是把这个新的类型处理器放到`Map`中，仅此而已。不过，值得注意的是，这个`Map`是一个两层嵌套结构，例子如下图所示。这也启示我们，如果要存储三元组，可以使用`Map`嵌套`Map`的方式。

![](https://i.loli.net/2020/03/21/ahi8fSrn53BZQ9w.png)