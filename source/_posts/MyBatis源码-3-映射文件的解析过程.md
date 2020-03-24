---
title: '[MyBatis源码][3][映射文件的解析过程]'
date: 2020-03-24 11:13:07
tags:
    - MyBatis
categories:
    - MyBatis
---


## 第三章 映射文件解析

### 3.1 映射文件解析入口

**映射文件的解析**过程是**配置文件解析**过程的**一部分**， MyBatis会在解析配置文件的过程中对映射文件进行解析。解析逻辑封装在`mapperElement()`方法中，我们把这个方法作为本章的总入口方法。

这个方法主要对`<mappers>`的每个子节点，按照不同的类型和属性，进行不同方式的解析，具体分派过程我放在源代码的注释中。

```java
//id:3.0
//package org.apache.ibatis.builder.xml;
//XMLConfigBuilder
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            
            //一、解析<package>子节点
            if ("package".equals(child.getName())) {
                String mapperPackage = child.getStringAttribute("name");
                configuration.addMappers(mapperPackage);
            
            //二、解析<mapper>子节点
            } else {
                
                //获取<mapper>节点的resource、url、mapperClass属性
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");
                
                //根据读取到的属性哪几个为空，来选择不同的解析方式，总共有4条路
                
                //① 根据resource解析
                if (resource != null && url == null && mapperClass == null) {
                    
                    ErrorContext.instance().resource(resource);
                    InputStream inputStream = Resources.getResourceAsStream(resource);
                    
                    XMLMapperBuilder mapperParser 
                        = new XMLMapperBuilder(inputStream, 
                                             configuration, 
                                             resource,
                                             configuration.getSqlFragments());
                    
                    mapperParser.parse();
                
                //② 根据url解析    
                } else if (resource == null && url != null && mapperClass == null) {
                
                    ErrorContext.instance().resource(url);
                    InputStream inputStream = Resources.getUrlAsStream(url);
                    
                    XMLMapperBuilder mapperParser 
                        = new XMLMapperBuilder(inputStream,
                                               configuration,
                                               url,
                                               configuration.getSqlFragments());
                    
                    mapperParser.parse();
                
                //③ 根据mapperClass解析
                } else if (resource == null && url == null && mapperClass != null) {
                    
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    configuration.addMapper(mapperInterface);
                
                //④ 都没有就直接报错    
                } else {
                    throw new BuilderException("A mapper element may only"
                                               + "specify a url, resource or class," 
                                               + "but not more than one.");
                }
            }
        }
    }
}
```



从代码段`3.0`中，我们可以看出，对于某个节点的解析逻辑，主要放在`XMLMapperBuilder`的`parse()`方法中，这个`parse()`方法主要包含如下几步

1. `mapper`节点的具体解析过程
2. 将这个节点设置为已加载
3. 通过命名空间绑定`Mapper`接口
4. 处理各个未完成解析的节点

```java
//id:3.1
//package org.apache.ibatis.builder.xml;
//XMLMapperBuilder
//Mapper的解析过程
public void parse() {
    
	//如果配置文件没有被加载过，就开始对这个XMLMapperBuilder进行解析
    if (!configuration.isResourceLoaded(resource)) {
      	
        //① mapper节点的具体解析过程
        configurationElement(parser.evalNode("/mapper"));
      	
        //② 将资源设置为已经加载
        configuration.addLoadedResource(resource);
      	
        //③ 通过命名空间绑定Mapper接口
        bindMapperForNamespace();
    }
	
    //④ 解析各个未完成解析的节点
    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
}
```

### 3.2 解析映射文件

```java
//id:3.2
//package org.apache.ibatis.builder.xml;
//XMLMapperBuilder
//对于单个mapper节点的具体解析过程
private void configurationElement(XNode context) {
    try {
        //① 获得mapper命名空间
        String namespace = context.getStringAttribute("namespace");
        
        //② 判断命名空间是否为空，为空报异常
        if (namespace == null || namespace.equals("")) {
            throw new BuilderException("Mapper's namespace cannot be empty");
        }
        
        //③ 设置当前命名空间
        builderAssistant.setCurrentNamespace(namespace);
        
        //④ 解析<cache-ref>节点
        cacheRefElement(context.evalNode("cache-ref"));
        
        //⑤ 解析<cache>节点
        cacheElement(context.evalNode("cache"));
        
        //⑥ 解析所有<parameterMap>节点，parameterMap主要考虑到可能传入sql的参数过于复杂
        parameterMapElement(context.evalNodes("/mapper/parameterMap"));
        
        //⑦ 解析所有<resultMap>节点
        resultMapElements(context.evalNodes("/mapper/resultMap"));
        
        //⑧ 解析所有<sql>节点
        sqlElement(context.evalNodes("/mapper/sql"));
        
        //⑨ 解析所有<select|insert|update|delete>节点
        buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing Mapper XML. The XML location is '" 
                                   + resource 
                                   + "'. Cause: " 
                                   + e, e);
    }
}
```



从上述代码段，我们可以得知，`configurationElement()`主要对单个`<mapper>`节点的各个类型的子节点进行解析，包括`<cache>`、`<cache-ref>`、`<parameterMap>`、`<resultMap>`、`<sql>`、`<select>`、`<insert>`、`<update>`、`<delete>`等子节点。在本节的剩余部分，我们将挑选几个有特点的`<mapper>`节点的子节点进行解析。

#### 3.2.1 解析`<cache>`节点



MyBatis提供了一、二级缓存，其中一级缓存是`SqlSession`级别的，默认为开启状态。二级缓存配置在映射文件中，使用者需要显式配置才能开启。



下面的代码段给出了一个配置二级缓存的例子

```xml
<!--id:3.3-->
<cache 
       eviction="FIFO"
       flushInterval="60000"
       size="512"
       readOnly="true"/>
```



那么我们废话少说，从代码块`3.2`的22行`cacheElement(context.evalNode("cache"));`向下，具体分析`cacheElement()`方法的逻辑。

```java
//id:3.4
//package org.apache.ibatis.builder.xml;
//XMLMapperBuilder
private void cacheElement(XNode context) {
    if (context != null) {
        
        //① 取得<cache>节点的各个属性
        String type = context.getStringAttribute("type", "PERPETUAL");
        Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
        
        String eviction = context.getStringAttribute("eviction", "LRU");
        Class<? extends Cache> evictionClass =	
            typeAliasRegistry.resolveAlias(eviction);
        
        Long flushInterval = context.getLongAttribute("flushInterval");
        
        Integer size = context.getIntAttribute("size");
        
        boolean readWrite = !context.getBooleanAttribute("readOnly", false);
        
        boolean blocking = context.getBooleanAttribute("blocking", false);
        
        //② 取得各个节点的子节点配置，其实还是一些属性
        Properties props = context.getChildrenAsProperties();
        
        //③ 调用MapperBuilderAssistant这个助手类，构建一个新的缓存，并存入Configuration中
        builderAssistant.useNewCache(typeClass,
                                     evictionClass,
                                     flushInterval,
                                     size,
                                     readWrite,
                                     blocking,
                                     props);
    }
}
```

`cacheElement`方法的逻辑还是比较简单的，它从`<cache>`节点上读取各种配置，然后使用这些配置。调用`Mapper`建造助手来构建一个新的缓存。下面我们从这个代码段的27行`builderAssistant.useNewCache(typeClass,evictionClass,flushInterval,size,readWrite,blocking,props);`向下，具体分析缓存的构造过程。



```java
//id:3.5
//package org.apache.ibatis.builder;
//MapperBuilderAssistant
public Cache useNewCache(Class<? extends Cache> typeClass,
                         Class<? extends Cache> evictionClass,
                         Long flushInterval,
                         Integer size,
                         boolean readWrite,
                         boolean blocking,
                         Properties props){

    //① 使用CacheBuilder，它是一个Cache的建造者，来一步一步建造一个Cache
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();

    //② 将这个新建好的Cache加入Configuration的Cache池中存放
    configuration.addCache(cache);
    
    //③ 没啥用
    currentCache = cache;
    return cache;
}
```



在我们继续深入源代码之前，首先我想介绍一个Mybatis的`Cache`接口极其实现类的设计。它使用一个装饰器模式。首先`Cache`接口有一个普通实现，`PerpetualCache`，它仅提供最基本的缓存功能，如果还需要其他功能，就需要将这个类作为`delegate`，包装到`Cache`接口的其他装饰器，例如，若我们想让`Cache`具有日志功能，就使用`LoggingCache`。下图展示了`Cache`接口的大量实现。

![](https://i.loli.net/2020/03/21/AHKVodpUer1SyZ5.png)



在了解了`Cache`接口的设计后，我们从`3.5`代码段的第21行`CacheBuilder.build()`向下，看看`build()`具体做了什么工作

```java
//id:3.6
//package org.apache.ibatis.mapping;
//CacheBuilder
public Cache build() {
    
    //① 设置默认的缓存类型和缓存装饰器
    setDefaultImplementations();
    
    //② 通过反射的方式，根据Class类的实例implementation来选择Cache接口合适的实现类来创建Cache
    Cache cache = newBaseCacheInstance(implementation, id);
    
    //③ 根据用户定义，设置这个cache的属性
    setCacheProperties(cache);
    
    // issue #352, do not apply decorators to custom caches
    
    //④ 若这个cache是PropertualCache(Cache的默认实现，没有任何装饰器)，则为它配置默认装饰器
    if (PerpetualCache.class.equals(cache.getClass())) {
        for (Class<? extends Cache> decorator : decorators) {
            cache = newCacheDecoratorInstance(decorator, cache);
            setCacheProperties(cache);
        }
        cache = setStandardDecorators(cache);
    
    //⑤. 如果这个cache没有使用任何日志装饰器，则加一个日志装饰器
    } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
        cache = new LoggingCache(cache);
    }
    return cache;
}
```

第13行的`setCacheProperties(cache);`主要将`<cache>`节点的各个`<property>`子节点设置进`Cache`的具体实现中，这个没什么可分析的，跳过它。这里，我们继续分析23行的`cache = setStandardDecorators(cache);`看看`PerpetualCache`都会被设置哪些标准装饰器。

```java
//id:3.7
//package org.apache.ibatis.mapping;
//CacheBuilder
private Cache setStandardDecorators(Cache cache) {
    try {
        //① 将size属性设置进PerpetualCache实例中
        MetaObject metaCache = SystemMetaObject.forObject(cache);
        if (size != null && metaCache.hasSetter("size")) {
            metaCache.setValue("size", size);
        }
        
        //② 如果前面代码清单3.4传入的clearInterval不为空，则加一个ScheduledCache`
        if (clearInterval != null) {
            cache = new ScheduledCache(cache);
            ((ScheduledCache) cache).setClearInterval(clearInterval);
        }
        
        //③ 如果readWrite标志开启，也就是读写锁功能开启，则加一个SerializedCache
        if (readWrite) {
            cache = new SerializedCache(cache);
        }
        
        //④ 加一个LoggingCache
        cache = new LoggingCache(cache);
        
        //⑤ 加一个SynchronizedCache
        cache = new SynchronizedCache(cache);
        
        //⑥ 如果blocking标记开启，加一个BlockingCache
        if (blocking) {
            cache = new BlockingCache(cache);
        }
        //⑦ 返回这个装饰过后的cache
        return cache;
    } catch (Exception e) {
        throw new CacheException("...");
    }
}
```

结合上述代码，这里画一个图，来表示标准装饰器的装饰顺序

![](https://i.loli.net/2020/03/21/AWavRPJoXM6Tpt7.png)

#### 3.2.2 解析`<resultMap>`节点

对于`resultMap`，引用官方文档的一段话，来说明其强大作用。

> resultMap元素是MyBatis中最重要最强大的元素。它可以让你从90%的 JDBC ResultSet数据提取代码中解放出来，并在一些情形下允许你做一些JBC不支持的事情。实际上，在对复杂语句进行联合映射的时候，它很可能可以代替数千行的同等功能的代码。 ResultMap的设计思想是，简单的语句不需要明确的结果映射，而复杂一点的语句只需要描述它们的关系就行了。



在分析源代码之前我们使用一个复杂的`resultMap`的例子来展示它的强大功能

```xml
<!--id:3.8-->
<resulpMap type="com.edu.neu.pojo.Employee" id="employee">
    <constructor>
        <idArg column="id" property="id" javaType="int" jdbcType="INT"/>
        <arg column="real_name" property="realName"/>
        <arg column="email" property="email"/>
    </constructor>
	<id column="id" property="id"/>
    <result column="real_name" property="realName"/>
    <result column="sex" property="sex" typeHandler="com.edu.neu.Handler.SexTypeHandler"/>
    <result column="email" property="email"/>
    <association property="workCard" column="id" select="com.edu.neu.mapper.WorkCardMapper.getWorkCardByEmpId"/>
    <collection property="employeeTaskList" column="id" select="com.edu.neu.mapper.EmployeeTaskMapper.getEmployeeTaskByEmpId"/>
</resulpMap>
```

这个例子几乎把`<resultMap>`的子节点演示遍了。下面解释一下这些子节点的作用

- `constructor`元素：用来配置一个构造方法。MyBatis会根据这个配置找到合适的构造方法对这个类实例化
- `result`元素：配置的是POJO成员变量到SQL列的映射关系，`column`代表SQL列名，`property`代表属性名
- `id`元素：除了具有`result`元素的功能，还表示了哪个列是主键，其实就是唯一标识列，不一定非要是主键。
- `association`元素：用来配置一个一对一的级联，例如上述代码段：当使用这个`resultMap`时，还会顺便把`WorkCard`也根据`id`从数据库中取出
- `collection`元素：与`association`元素类似，也是完成级联，但是它用于一对多级联，它将返回一个`java.util.List`
- 其他子节点并不常见，这里就不介绍了。



在正式分析解析逻辑之前，我们先看看存储结构，所有解析完成的`ResultMap`都将存放在`Configuration`的成员变量`resultMaps`中，这个`Map`的键为我们为`<resultMap>`节点指定的`id`属性，例如代码清单`3.8`的`employee`就将成为这个`resultMap`的`key`。而值为一个`org.apache.ibatis.mapping.ResultMap`的实例，这个类存储了单个`<resultMap>`的解析结果。

```java
//id:3.9
//package org.apache.ibatis.session;
//Configuration
protected final Map<String, ResultMap> resultMaps 
      = new StrictMap<>("Result Maps collection");
```



下面我们再看看上文中提到的`org.apache.ibatis.mapping.ResultMap`中都存放了什么吧

```java
//id:3.10
//package org.apache.ibatis.mapping;
//ResultMap
public class ResultMap {
    
    //一个configuration的引用，主要用来操作configuration来存放解析结果
  	private Configuration configuration;
    
    //这个<resultMap>的id属性，唯一标识符
	private String id;
    
    //这个<resultMap>的type属性，被解析为了一个Class对象
  	private Class<?> type;
    
    //用于存放<resultMap>的各个<result>、<id>节点、以及<contructor>中的<idArg>、<arg>节点的解析结果
  	private List<ResultMapping> resultMappings;
  	
    //用于存放<resultMap>中<id>节点的解析结果
    private List<ResultMapping> idResultMappings;
    
    //用来存放<resultMap>中<constructor>节点的所有解析结果
  	private List<ResultMapping> constructorResultMappings;
    
    //用于存放<resultMap>中<result>节点的解析结果
  	private List<ResultMapping> propertyResultMappings;
    
    //用来存放SQL表所有被映射的列的列名
  	private Set<String> mappedColumns;
    
    //用来存放POJO所有被映射的属性的属性名
  	private Set<String> mappedProperties;
    
    //一个鉴别器，不常用，不分析
  	private Discriminator discriminator;
    
    //一个标志，是否有嵌套的ResultMap
  	private boolean hasNestedResultMaps;
    
    //一个标志，是否有嵌套的查询
  	private boolean hasNestedQueries;
    
    //一个标志，是否开启了自动映射
  	private Boolean autoMapping;
	
    //....
}
```

其实，说白了，就是把单个`<result>`、`<id>`的解析结果，按照不同的类型，在不同的`List`中存放了起来，仅此而已。下图是个很好的例子。

![](https://i.loli.net/2020/03/22/AOcWNLmzvPgh9SY.png)



但是这还没完，上述代码块用到的`ResultMapping`类，它看起来是存储单个POJO-SQL映射的类，我们接着分析它。

```java
//id:3.11
//package org.apache.ibatis.mapping;
//ResultMapping
public class ResultMapping {
	//一个configuration的引用
    private Configuration configuration;
  	//这个节点的property属性，对应POJO的属性
    private String property;
  	//这个节点的column属性，对应SQL的列名
    private String column;
  	//POJO属性的JavaType，java类型
    private Class<?> javaType;
  	//SQL列对应的JDBCType，JDBC类型
    private JdbcType jdbcType;
  	//用来处理这个javaType和这个JDBCType的互相转换的类型处理器
    private TypeHandler<?> typeHandler;
  	//略
    private String nestedResultMapId;
  	//略
    private String nestedQueryId;
  	//略
    private Set<String> notNullColumns;
  	//略
    private String columnPrefix;
  	//标志这个列是否为主键、或者是否在`<contructor>`中存在等
    //ResultFlag就是一个简单的枚举类，这没啥可说的
    private List<ResultFlag> flags;
  	//略
    private List<ResultMapping> composites;
  	//略
    private String resultSet;
  	//略
    private String foreignColumn;
  	//略
    private boolean lazy;
}
```



那么`resultMap`的存储结构就分析完了，我们继续看源码，从代码清单3.2的28行`        resultMapElements(context.evalNodes("/mapper/resultMap"));`向下，详细分析`resultMap`的解析过程

```java
//id:3.12
//package org.apache.ibatis.builder.xml;
//XMLMapperBuilder
private void resultMapElements(List<XNode> list) throws Exception {
    for (XNode resultMapNode : list) {
        try {
            resultMapElement(resultMapNode);
        } catch (IncompleteElementException e) {
            // ignore, it will be retried
        }
    }
}

private ResultMap resultMapElement(XNode resultMapNode) throws Exception {
    return resultMapElement(resultMapNode, Collections.emptyList(), null);
}

private ResultMap resultMapElement(XNode resultMapNode, 
                                   List<ResultMapping> additionalResultMappings,
                                   Class<?> enclosingType) throws Exception {
    
    ErrorContext.instance().activity("processing " +
                                     resultMapNode.getValueBasedIdentifier());
    
    //从<resultMap>节点上读取属性值type
    //若type不存在就寻找ofType，以此类推地寻找resultType和javaType
    String type = resultMapNode.getStringAttribute("type", 
                resultMapNode.getStringAttribute("ofType",
                resultMapNode.getStringAttribute("resultType",
                resultMapNode.getStringAttribute("javaType"))));
    
    //根据找到的type字符串，生成type对应的Class对象
    Class<?> typeClass = resolveClass(type);
    
    //略
    if (typeClass == null) {
      typeClass = inheritEnclosingType(resultMapNode, enclosingType);
    }
    
    //略
    Discriminator discriminator = null;
    
    //ResultMapping负责存在单个pojo-Sql映射，比如<id>、<result>节点中包含的映射信息
    List<ResultMapping> resultMappings = new ArrayList<>();
    resultMappings.addAll(additionalResultMappings);
    List<XNode> resultChildren = resultMapNode.getChildren();
    
    //遍历这个<resultMap>的所有子节点
    for (XNode resultChild : resultChildren) {
        
        //处理<constructor>节点
        if ("constructor".equals(resultChild.getName())) {
            processConstructorElement(resultChild, typeClass, resultMappings);
        
        //处理<discriminator>节点    
        } else if ("discriminator".equals(resultChild.getName())) {
            discriminator 
                = processDiscriminatorElement(resultChild, typeClass, resultMappings);
        
        //处理其他节点    
        } else {
            
            List<ResultFlag> flags = new ArrayList<>();
            
            //如果这个节点是<id>，则添加一个主键标志
            if ("id".equals(resultChild.getName())) {
                flags.add(ResultFlag.ID);
            }
            
            //<id>、<result>等节点的具体解析过程
            resultMappings.
                add(buildResultMappingFromContext(resultChild, typeClass, flags));
        }
    }
    
    //获取这个<resultMap>的id属性
    String id = resultMapNode.getStringAttribute("id",
                resultMapNode.getValueBasedIdentifier());
    
    //获取这个<resultMap>的extend属性
    String extend = resultMapNode.getStringAttribute("extends");
    
    //获取这个<resultMap>是否开启自动映射
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    
    //根据前面获取的各种信息创建ResultMap解析器
    ResultMapResolver resultMapResolver 
        = new ResultMapResolver(builderAssistant, id, typeClass, extend,
                                discriminator, resultMappings, autoMapping);
    
    try {
        //通过这些信息，解析并返回ResultMap对象
        return resultMapResolver.resolve();

    } catch (IncompleteElementException  e) {
    	//如果解析出错了就假如到Configuration的未成功解析列表中
        configuration.addIncompleteResultMap(resultMapResolver);
        throw e;
    }
}
```

上述`resultMapElement`解析过程还是挺复杂的，这里总结一下，它完成的几项工作

1. 获取`<resultMap>`节点的各种属性
2. 解析`<resultMap>`的所有子节点，并把返回结果存起来
3. 用第1步和第2步获取的信息构造一个`ResultMap`对象
4. 若第3步构造失败，则添加到未成功解析列表并抛出异常

第1步比较简单，大家一看就懂。第2步将在3.2.2.1中展开分析。第3步将在3.2.2.2中展开分析。

##### 3.2.2.1 解析`<resultMap>`节点中的`<id>`节点和`<result>`节点

本节以`<id>`节点和`<result>`节点为例，分析`<resultMap>`节点的子节点是如何解析的。那我们从代码清单`3.12`的71行`resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));`向下。

```java
//id:3.13
//package org.apache.ibatis.builder.xml;
//XMLMapperBuilder
private ResultMapping buildResultMappingFromContext(
    XNode context, Class<?> resultType, List<ResultFlag> flags) throws Exception {
    
    String property;
    
    if (flags.contains(ResultFlag.CONSTRUCTOR)) {
      	property = context.getStringAttribute("name");
    } else {
      	property = context.getStringAttribute("property");
    }
    
    String column = context.getStringAttribute("column");
    
    String javaType = context.getStringAttribute("javaType");
    
    String jdbcType = context.getStringAttribute("jdbcType");
    
    String nestedSelect = context.getStringAttribute("select");
    
    String nestedResultMap = context.getStringAttribute("resultMap",
        processNestedResultMappings(context, Collections.emptyList(), resultType));
    
    String notNullColumn = context.getStringAttribute("notNullColumn");
    
    String columnPrefix = context.getStringAttribute("columnPrefix");
    
    String typeHandler = context.getStringAttribute("typeHandler");
    
    String resultSet = context.getStringAttribute("resultSet");
    
    String foreignColumn = context.getStringAttribute("foreignColumn");
    
    boolean lazy = "lazy".equals(
        context.getStringAttribute(
            "fetchType", configuration.isLazyLoadingEnabled() ? "lazy" : "eager"));
    
    Class<?> javaTypeClass = resolveClass(javaType);
    
    Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
    
    JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
    
    return builderAssistant.buildResultMapping(
        resultType,
        property,
        column,
        javaTypeClass,
        jdbcTypeEnum,
        nestedSelect,
        nestedResultMap,
        notNullColumn,
        columnPrefix,
        typeHandlerClass,
        flags,
        resultSet,
        foreignColumn,
        lazy);
  }
```

这个方法实在乏善可陈，它只是从`<result>`或者`<id>`节点上获取了各种属性，然后将这些获取到的属性统统传递给` builderAssistant.buildResultMapping()`，然后这个助手类完成真正的解析工作并返回`ResultMapping`对象

```java
//id:3.13
//package org.apache.ibatis.builder;
//MapperBuilderAssistant
public ResultMapping buildResultMapping(
      Class<?> resultType,
      String property,
      String column,
      Class<?> javaType,
      JdbcType jdbcType,
      String nestedSelect,
      String nestedResultMap,
      String notNullColumn,
      String columnPrefix,
      Class<? extends TypeHandler<?>> typeHandler,
      List<ResultFlag> flags,
      String resultSet,
      String foreignColumn,
      boolean lazy) {
    
    Class<?> javaTypeClass = 
        resolveResultJavaType(resultType, property, javaType);
    
    TypeHandler<?> typeHandlerInstance = 
        resolveTypeHandler(javaTypeClass, typeHandler);
    
    List<ResultMapping> composites = parseCompositeColumnName(column);
    
    return new ResultMapping.Builder(configuration, property, column, javaTypeClass)
        .jdbcType(jdbcType)
        .nestedQueryId(applyCurrentNamespace(nestedSelect, true))
        .nestedResultMapId(applyCurrentNamespace(nestedResultMap, true))
        .resultSet(resultSet)
        .typeHandler(typeHandlerInstance)
        .flags(flags == null ? new ArrayList<>() : flags)
        .composites(composites)
        .notNullColumns(parseMultipleColumnNames(notNullColumn))
        .columnPrefix(columnPrefix)
        .foreignColumn(foreignColumn)
        .lazy(lazy)
        .build();
}
```

至于这个`ResultMapping.Builder`就不继续深入了，它是一个简单的建造者负责建造`ResultMapping`。其实就是通过各种方法调用，为`ResultMapping`的实例设置属性而已。设置完成后，调用`build()`直接返回这个实例。



下面总结一下，对于`<id>`和`<result>`这种代表单个POJO-SQL映射的标签，MyBatis会将标签携带的属性进行解析，并全部存放在一个`ResultMapping`实例中返回。



##### 3.2.2.2 构建`ResultMap`对象的过程

我们从代码清单`3.12`的93行`return resultMapResolver.resolve();`，看看这个解析器是如何构建`ResultMap`的

```java
//id:3.13
//package org.apache.ibatis.builder;
//ResultMapResolver
public ResultMap resolve() {
    return assistant.addResultMap(this.id,
                                  this.type,
                                  this.extend,
                                  this.discriminator,
                                  this.resultMappings,
                                  this.autoMapping);
}
```



代码清单`3.13`实际上调用了建造器助手的`addResultMap`方法，我们继续向下

```java
//id:3.14
//package org.apache.ibatis.builder;
//MapperBuilderAssistant
public ResultMap addResultMap(String id,
                              Class<?> type,
                              String extend,
                              Discriminator discriminator,
                              List<ResultMapping> resultMappings,
                              Boolean autoMapping) {

    id = applyCurrentNamespace(id, false);
    
    extend = applyCurrentNamespace(extend, true);

    //处理继承情况，不展开了
    if (extend != null) {
        if (!configuration.hasResultMap(extend)) {
            throw new IncompleteElementException(
                "Could not find a parent resultmap with id '" 
                + extend + "'");
        }
        ResultMap resultMap 
            = configuration.getResultMap(extend);
        
        List<ResultMapping> extendedResultMappings 
            = new ArrayList<>(resultMap.getResultMappings());
        
        extendedResultMappings.removeAll(resultMappings);
        
        // Remove parent constructor if this resultMap declares a constructor.
        boolean declaresConstructor = false;
        for (ResultMapping resultMapping : resultMappings) {
            if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
                declaresConstructor = true;
                break;
            }
        }
        
        if (declaresConstructor) {
            extendedResultMappings.removeIf(
                resultMapping -> resultMapping.
				getFlags().contains(ResultFlag.CONSTRUCTOR)
            );
        }
        resultMappings.addAll(extendedResultMappings);
    }

    //使用建造者构造ResultMap
    ResultMap resultMap = 
        new ResultMap.Builder(configuration,
                              id,
                              type,
                              resultMappings,
                              autoMapping)
      						.discriminator(discriminator)
      						.build();
    
    //将得到的ResultMap保存到configuration的resultMaps中
    configuration.addResultMap(resultMap);
    
    return resultMap;
}
```

这个方法实际上做了这几件事

1. 处理`resultMap`的继承(`extend`属性)
2. 通过`ResultMap`的建造者构造`ResultMap`实例
3. 将这个`ResultMap`实例保存到`configuration`的`resultMaps`中



下面我们继续看看这个建造者是怎么完成建造工作的。

```java
//id:3.15
//package org.apache.ibatis.mapping;
//ResultMap.Builder
public ResultMap build() {
    //如果这个resultMap没有id，也就是唯一标识符，直接报错
    if (resultMap.id == null) {
      throw new IllegalArgumentException("ResultMaps must have an id");
    }
    
    //把这个resultMap需要使用但没有传入的List和Set初始化
    resultMap.mappedColumns = new HashSet<>();
    resultMap.mappedProperties = new HashSet<>();
    resultMap.idResultMappings = new ArrayList<>();
    resultMap.constructorResultMappings = new ArrayList<>();
    resultMap.propertyResultMappings = new ArrayList<>();
    
    //初始化构造器参数名列表
    final List<String> constructorArgNames = new ArrayList<>();
    
    //遍历所有Mapping
    for (ResultMapping resultMapping : resultMap.resultMappings) {
        
        //设置<association>和<collection>标记
        resultMap.hasNestedQueries = 
            resultMap.hasNestedQueries || resultMapping.getNestedQueryId() != null;
        //同上
        resultMap.hasNestedResultMaps = 
            resultMap.hasNestedResultMaps ||
            (resultMapping.getNestedResultMapId() != null
             && resultMapping.getResultSet() == null);
        
        //获取SQL列名
        final String column = resultMapping.getColumn();
        
        //将列名加入mappedColumns集合
        //列名不为空的情况
        if (column != null) {
            resultMap.mappedColumns.add(column.toUpperCase(Locale.ENGLISH));
        
        //Mapping是复合结构的情况
        } else if (resultMapping.isCompositeResult()) {
            
            for (ResultMapping compositeResultMapping :
                 resultMapping.getComposites()){
                
                final String compositeColumn = compositeResultMapping.getColumn();
                if (compositeColumn != null) {
               		resultMap.mappedColumns.
                        add(compositeColumn.toUpperCase(Locale.ENGLISH));
                }
            }
        }
      	
        //获取POJO属性名
        final String property = resultMapping.getProperty();
    
        //将属性名加入mappedProperties集合
        if (property != null) {
            resultMap.mappedProperties.add(property);
        }
      
        //将带有构造器标记的mapping加入constructorResultMappings列表
        if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
            resultMap.constructorResultMappings.add(resultMapping);
            if (resultMapping.getProperty() != null) {
                constructorArgNames.add(resultMapping.getProperty());
            }
        
        //将不带构造器标记的mapping加入propertyResultMappings列表  
        } else {
            resultMap.propertyResultMappings.add(resultMapping);
        }
        
        //将带有主键标记的mapping加入idResultMappings列表
        if (resultMapping.getFlags().contains(ResultFlag.ID)) {
            resultMap.idResultMappings.add(resultMapping);
        }
    }
    
    //如果这个resultMap没有主键，就让所有人做主键
    if (resultMap.idResultMappings.isEmpty()) {
        resultMap.idResultMappings.addAll(resultMap.resultMappings);
    }
    
    //如果前面获取的参数名列表不为空，则通过反射按照这个列表获取参数的实际名列表
    //并按照获取的参数实际名称列表对constructorResultMappings进行排序(也就是传参要按顺序)
    if (!constructorArgNames.isEmpty()) {
        final List<String> actualArgNames = 
            argNamesOfMatchingConstructor(constructorArgNames);
        if (actualArgNames == null) {
            throw new BuilderException("Error in result map '" + resultMap.id
                + "'. Failed to find a constructor in '"
                + resultMap.getType().getName() + "' by arg names " 
                + constructorArgNames
                + ". There might be more info in debug log.");
        }
        resultMap.constructorResultMappings.sort((o1, o2) -> {
            int paramIdx1 = actualArgNames.indexOf(o1.getProperty());
            int paramIdx2 = actualArgNames.indexOf(o2.getProperty());
            return paramIdx1 - paramIdx2;
        });
    }
    
    //将resultMappings等一干集合类冻结为不能修改的状态
    // lock down collections
    resultMap.resultMappings 
        = Collections.unmodifiableList(resultMap.resultMappings);
    resultMap.idResultMappings 
        = Collections.unmodifiableList(resultMap.idResultMappings);
    resultMap.constructorResultMappings 
        = Collections.unmodifiableList(resultMap.constructorResultMappings);
    resultMap.propertyResultMappings 
        = Collections.unmodifiableList(resultMap.propertyResultMappings);
    resultMap.mappedColumns 
        = Collections.unmodifiableSet(resultMap.mappedColumns);
    return resultMap;
}
```

上面的代码比较长，但实际上就如代码清单`3.10`所示，它将传入的`resultMappings`列表中的元素，按照不同的特点放入了不同的列表和集合中，仅此而已。



到此，我们就完成了`ResultMap`对象的构建，并且将构建完的结果以`id`做键、`ResultMap`做值的形式存放到了configuration的`resultMaps`映射中。本节比较值得学习的就是MyBatis对于建造者模式的使用。



#### 3.2.3 解析`<sql>`节点

`<sql>`节点用来定义一些可重用的SQL语句片段，比如表名，或表的列名等。在映射文件中，我们可以通过 `<include>`节点引用`<sql>`节点定义的内容。



在分析源码之前，先来演示一下`<sql>`节点的使用方式

```xml
<!--id:3.16-->
<sql id="table">
	article
</sql>

<select id="findOne" resultType="Article">
	SELECT id, title 
    FROM <include refid="table"/> 
    WHERE id = #{id}
</select>

<update id="update" parameterType="Article">
	UPDATE <include refid="table"/> 
    SET title = #{title} 
    WHERE id = #{id}
</update>
```



然后，我们从代码清单`3.2`的32行`sqlElement(context.evalNodes("/mapper/sql"));`继续向下，看看对于`<sql>`节点的解析。

```java
//id:3.17
//package org.apache.ibatis.builder.xml;
//XMLMapperBuilder

//用于存放解析完毕的<sql>节点，从configuration中取得的
private final Map<String, XNode> sqlFragments;

//代码清单3.2的32行调用此方法
//它先解析所有databaseId与当前数据库匹配的<sql>节点
//然后解析所有不带databaseId的<sql>节点
private void sqlElement(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
        sqlElement(list, configuration.getDatabaseId());
    }
    sqlElement(list, null);
}

private void sqlElement(List<XNode> list, String requiredDatabaseId) {
    //遍历所有<sql>节点
    for (XNode context : list) {
        
        //获取databaseId
        String databaseId = context.getStringAttribute("databaseId");
        //获取这个<sql>节点的id属性
        String id = context.getStringAttribute("id");
        
        //在这个id属性的前面加上当前的命名空间
        //例如:table -> com.edu.neu.zady.dao.ProductDao.table
        id = builderAssistant.applyCurrentNamespace(id, false);
        
        //如果数据库id符合，则将这个<sql>节点直接添加到sqlFragment映射中
        if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
            sqlFragments.put(id, context);
        }
    }
}
```

`<sql>`节点的解析非常简单，它只不过是完成了以下几件事

1. 通过`databaseId`筛选符合当前数据库的`<sql>`节点

2. 将符合要求的节点加入`sqlFragment`映射，这个映射将在解析SQL语句节点时使用

   

并且，其实这个`sqlFragement`也是存储在`Configuration`中的，方便后面的使用。



#### 3.2.4 解析SQL语句节点



下面是本章的重头戏，`<select>`、`<insert>`、`<update>`、`<delete>`等SQL语句节点的解析。这些节点的用处都是存储SQL语句，所以解析过程是相同的。



在分析之前，我们还是先看看在`Configuration`中解析完的信息是怎么储存的。对于每个SQL语句节点，MyBatis都会解析成一个`MappedStatement`的实例。然后在`Configuration`中，是通过以`id`为键，以`MappedStatement`本身为值存储在了一个`Map`中。

```java
protected final Map<String, MappedStatement> mappedStatements 
    = new StrictMap<MappedStatement>("Mapped Statements collection");
```



接下来我们看看，上面提到的`MappedStatement`都存储了那些信息

```java
//id:3.18
//package org.apache.ibatis.mapping;
//MappedStatement
public final class MappedStatement {
    //略
    private String resource;
    //一个Configuration的引用
    private Configuration configuration;
    //略
    private String id;
    //略
    private Integer fetchSize;
    //略
    private Integer timeout;
    //STATEMENT, PREPARED, CALLABLE
    private StatementType statementType;
    //DEFAULT, FORWARD_ONLY, SCROLL_INSENSITIVE, SCROLL_SENSITIVE;
    private ResultSetType resultSetType;
    //存放具体的SQL语句，还有参数列表等
    private SqlSource sqlSource;
    //这个Statement使用的二级缓存
    private Cache cache;
    //存放使用的参数
    private ParameterMap parameterMap;
    //存放使用的一些ResultMap
    private List<ResultMap> resultMaps;
    //略
    private boolean flushCacheRequired;
    //略
    private boolean useCache;
    //略
    private boolean resultOrdered;
    //UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH
    private SqlCommandType sqlCommandType;
    //用来自增主键
    private KeyGenerator keyGenerator;
    //略
    private String[] keyProperties;
    //略
    private String[] keyColumns;
    //略
    private boolean hasNestedResultMaps;
    //略
    private String databaseId;
    //略
    private Log statementLog;
    //略
    private LanguageDriver lang;
    //略
    private String[] resultSets;
}
```



让我们详细展开上一个代码清单的20行`private SqlSource sqlSource;`看看`SqlSource`是什么

```java
//id:3.18
//package org.apache.ibatis.mapping;
//SqlSource
public interface SqlSource {

  BoundSql getBoundSql(Object parameterObject);

}
```

它是个接口，这个接口传入`parameter`，然后返回一个`BoundSql`实例。那我们接着看看`BoundSql`的结构



```java
//id:3.19
//package org.apache.ibatis.mapping;
//BoundSql
public class BoundSql {
	//真.SQL语句
  	private final String sql;
  	//参数的列表
    private final List<ParameterMapping> parameterMappings;
  	//略
    private final Object parameterObject;
  	//额外参数
    private final Map<String, Object> additionalParameters;
  	//参数的元数据信息
    private final MetaObject metaParameters;
}
```

至此，和SQL语句有关的存储结构算是分析完了。



我们下面从代码清单`3.2`的34行`buildStatementFromContext(context.evalNodes("select|insert|update|delete"));`继续向下，看看`MappedStatement`是怎么构建的

```java
//id:3.20
//package org.apache.ibatis.builder.xml;
//XMLMapperBuilder
private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
      buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
}

private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    //遍历每个<select|insert|update|delete>节点
    for (XNode context : list) {
        final XMLStatementBuilder statementParser = 
            new XMLStatementBuilder(configuration,
                                    builderAssistant,
                                    context,
                                    requiredDatabaseId);
        try {
            //每个节点的实际解析逻辑
            statementParser.parseStatementNode();
        } catch (IncompleteElementException e) {
            configuration.addIncompleteStatement(statementParser);
        }
    }
}
```

这个方法其实什么也没干，它制作遍历每个节点，然后把具体每个节点的解析交给`XMLstatementBuilder`的`parseStatementNode()`来处理，具体的解析逻辑都在这个方法里，那我们继续向下看看这个方法。



在看源码之前，先大体描述这个方法进行的几步操作

1. 解析SQL语句中的`<include>`节点，第34行
2. 解析SQL语句中的`<selectKey>`节点，第46行
3. 解析SQL语句， 第67行
4. 构建`MappedStatement`，第93行



```java
//id:3.20
//package org.apache.ibatis.builder.xml;
//XMLStatementBuilder
public void parseStatementNode() {
    
    //获取<select|insert|update|delete>节点的id
    String id = context.getStringAttribute("id");
    
    //获取<select|insert|update|delete>节点的dataBaseId
    String databaseId = context.getStringAttribute("databaseId");

    //如果dataBaseId与当前数据库不匹配，则不解析这个节点，直接返回
    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }

    //获取这个节点的名称，select、insert、update还是delete
    String nodeName = context.getNode().getNodeName();
    
    //通过节点的名称创建一个SqlCommandType
    SqlCommandType sqlCommandType 
        = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    
    //根据节点的属性来设置一些标志位
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

    //解析SQL语句中的<include>节点
    //例如代码清单3.16的第8行、第13行
    // Include Fragments before parsing
    XMLIncludeTransformer includeParser 
        = new XMLIncludeTransformer(configuration, builderAssistant);
    
    //同上
    includeParser.applyIncludes(context.getNode());

    //获取节点的parameterType属性
    String parameterType = context.getStringAttribute("parameterType");
    Class<?> parameterTypeClass = resolveClass(parameterType);

    //获取节点的语言属性
    String lang = context.getStringAttribute("lang");
    LanguageDriver langDriver = getLanguageDriver(lang);

    //解析<selectKey>节点
    // Parse selectKey after includes and remove them.
    processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    KeyGenerator keyGenerator;
    
    //根据命名空间等信息，为这个Statement(也就是这个<select|update|delete|insert>节点)起名
    String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    
    //处理id重名的情况
    if (configuration.hasKeyGenerator(keyStatementId)) {
      keyGenerator = configuration.getKeyGenerator(keyStatementId);
    } else {
      keyGenerator = 
          context.getBooleanAttribute("useGeneratedKeys",
          configuration.isUseGeneratedKeys() &&
                                      SqlCommandType.INSERT.equals(sqlCommandType))
          ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    }

    //生成sqlSource
    SqlSource sqlSource =
        langDriver.createSqlSource(configuration, context, parameterTypeClass);
    
    //生成statementType，比如:Statement、PreparedStatement、CallableStatement
    StatementType statementType =
        StatementType.valueOf(context.getStringAttribute(
            "statementType", StatementType.PREPARED.toString()));
    
   	//继续获取一些属性 
    Integer fetchSize = context.getIntAttribute("fetchSize");
    Integer timeout = context.getIntAttribute("timeout");
    String parameterMap = context.getStringAttribute("parameterMap");
    String resultType = context.getStringAttribute("resultType");
    Class<?> resultTypeClass = resolveClass(resultType);
    String resultMap = context.getStringAttribute("resultMap");
    String resultSetType = context.getStringAttribute("resultSetType");
    ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    String keyProperty = context.getStringAttribute("keyProperty");
    String keyColumn = context.getStringAttribute("keyColumn");
    String resultSets = context.getStringAttribute("resultSets");

    //将前面获取的各种信息，全都传递给builderAssistant.addMappedStatement方法，
    //这个方法将完成MappedStatement的生成操作，并添加到Configuration中
    builderAssistant.addMappedStatement(
        id,
        sqlSource,
        statementType,
        sqlCommandType,
        fetchSize,
        timeout,
        parameterMap,
        parameterTypeClass,
        resultMap,
        resultTypeClass,
        resultSetTypeEnum,
        flushCache,
        useCache, 
        resultOrdered,
        keyGenerator,
        keyProperty,
        keyColumn,
        databaseId,
        langDriver,
        resultSets);
}
```

下面的4小节，将分别展开这四个步骤



##### 3.2.4.1 解析`<include>`节点



注：下面的解析过程比较难，看不懂可以先看后面的例子。如果实在看不懂，这里讲一下这方法执行后的结果。它将`XNODE`树上的`<include>`节点替换成了包含对应`sql`语句的普通文本节点。也就是说，经过这一步的处理，`<include>`节点在`XNODE`树中消失了。我们在`mapper`文件的层次上举个不太恰切的例子。



在没进行解析时，`XNODE`树是这样的

```xml
<!--id:3.21-->
<mapper namespace="xyz.coolblog.dao.ArticleDao">
	
    <sql id="table">
		${table_name}
	</sql>
	
    <select id="findOne" resultType="xyz.coolblog.dao.Article">
		SELECT id, title
		FROM 
        <include refid="table">
			<property name="table_name" value="article"/>
		</include>
		WHERE id = #{id}
	</select>

</mapper>
```

完成解析之后，它变成了一个再普通不过的sql

```xml
<!--id:3.22-->
<mapper namespace="xyz.coolblog.dao.ArticleDao">
	
    <sql id="table">
		${table_name}
	</sql>
	
    <select id="findOne" resultType="xyz.coolblog.dao.Article">
		SELECT id, title
		FROM article
		WHERE id = #{id}
	</select>

</mapper>
```

只不过上面的解析，不是在mapper文件的层面上进行的，而是在`XNODE`的层面进行的。大家体会理解意思即可。



然后我们从代码清单`3.20`第37行`includeParser.applyIncludes(context.getNode());`向下，看看`<include>`节点的解析过程。

```java
//id:3.21
//package org.apache.ibatis.builder.xml;
//XMLIncludeTransformer
//这个方法不要按顺序读，看不懂先看看后面的例子
private void applyIncludes(Node source, 
                           final Properties variablesContext,
                           boolean included) {
    
    //第一个分支，用于处理<include>节点
    if (source.getNodeName().equals("include")) {
    	//获取<sql>节点，如果refid中包含属性占位符${}
        //则需先将属性占位符替换为对应的属性
        Node toInclude =
            findSqlFragment(getStringAttribute(source, "refid"), variablesContext);
        
        //获得<include>节点的所有<property>子节点，并将结果与variablesContext混合
        Properties toIncludeContext = getVariablesContext(source, variablesContext);
        
        //递归调用，对<sql>节点执行applyIncludes，替换其中的${}
        applyIncludes(toInclude, toIncludeContext, true);
        
        //处理<sql>节点在其他文件的情况
        if (toInclude.getOwnerDocument() != source.getOwnerDocument()) {
            toInclude = source.getOwnerDocument().importNode(toInclude, true);
        }
        
        //将<include>节点替换为<sql>节点
        source.getParentNode().replaceChild(toInclude, source);
        
        //将解析完成的<sql>节点里的Text内容插入到<sql>节点之前
        while (toInclude.hasChildNodes()) {
            toInclude.getParentNode().
                insertBefore(toInclude.getFirstChild(), toInclude);
        }
        
        //前面插入的Text内容节点是解析好的，已经可以完全代替<sql>节点了
        //那么我们直接将<sql>节点也移除掉
        toInclude.getParentNode().removeChild(toInclude);
        
    //第二个条件分支，用来处理<select|insert|update|delete>节点或者<sql>节点
    //总之就是除了<include>节点之外的所有普通节点
    } else if (source.getNodeType() == Node.ELEMENT_NODE) {
        
        if (included && !variablesContext.isEmpty()) {
        // replace variables in attribute values
        	
            //获取这个节点的所有属性
        	NamedNodeMap attributes = source.getAttributes();
        
            //遍历这些属性，将属性中的${}替换为具体的值
        	for (int i = 0; i < attributes.getLength(); i++) {
          		Node attr = attributes.item(i);
          		attr.setNodeValue(PropertyParser.parse(
                  	attr.getNodeValue(), variablesContext));
        	}
    	}
        
        //获取这个节点的所有子节点
        NodeList children = source.getChildNodes();
        
        //对每个子节点执行applyIncludes
        for (int i = 0; i < children.getLength(); i++) {
            applyIncludes(children.item(i), variablesContext, included);
        }
    
    //处理所有TEXT节点    
    } else if (
        included 
        && source.getNodeType() == Node.TEXT_NODE 
        && !variablesContext.isEmpty()) {
      	// replace variables in text node
        //将节点中的${}替换为具体的值
        source.setNodeValue(
            PropertyParser.parse(
                source.getNodeValue(), variablesContext));
    }
}
```



然后我们以代码清单`3.21`从8-12行的`<select>`节点的解析为例，详细看看`<include>`是怎么被替换的。



但是这里还要插入一个先序知识：在`XNODE`树中，所有的`<xxx>`标签会被解析为`ELEMENT_NODE`，而所有`<xxx>`和`</xxx>`间的文本将被解析为`TEXT_NODE`，除此之外还有`ATTRIBUTE_NODE`、`COMMENT_NODE`等很多节点类型，有兴趣可以查看`org.w3c.dom.Node`接口



那么这个`<select>`节点的类型为`ELEMENT_NODE`，它有三个子节点，如下表

| 编号 | 子节点                     | 类型           | 描述     |
| ---- | -------------------------- | -------------- | -------- |
| 1    | `SELECT id,title FROM`     | `TEXT_NODE`    | 文本节点 |
| 2    | `<include refid="table"/>` | `ELEMENT_NODE` | 普通节点 |
| 3    | `WHERE id= #{id}`          | `TEXT_NODE`    | 文本节点 |



那么调用的入口代码清单`3.20`第34行`includeParser.applyIncludes(context.getNode());`传入的`XNode`显然是`<select>`节点。它会进入第二个条件，遍历自己的3个孩子节点。第一个节点和第二个节点的调用栈如下图

![](https://i.loli.net/2020/03/22/Js3PGbzmCI6yRux.png)

![](https://i.loli.net/2020/03/22/pwUle9y8631tfkr.png)

##### 3.2.4.2 解析`<selectKey>`节点



对于一些不支持自增主键的数据库来说，我们在插入数据时，需要明确指定主键数据。例如

```xml
<!--id:3.22-->
<insert id="saveAuthor">
	<selectKey keyProperty="id" resultType="int" order="BEFORE">
		select author_seq.nextval from dual
	</selectKey>
	insert into Author
		(id, name, password)
	values
		(#{id}, #{username}, #{password})
</insert>
```

这部分的源码就不展开解析了。当Mybatis完成解析后，也会将`<selectKey>`节点从`XNODE`树中去掉



##### 3.2.4.3 解析SQL语句生成`SqlSource`



经过上两节的解析，MyBatis已经把`<select|insert|delete|create>`中所有的`<include>`和`<selectKey>`子孙节点全部都替换并删除掉了。现在`XNODE`树中只有`<if>`、`<where>`等普通的`ELEMENT`节点和文本节点。这一步，我们将分析MyBatis是如何解析`<select|insert|delete|create>`节点的`XNODE`树，来生成`SqlSource`。当处理用户的实际调用时，MyBatis将通过`SqlSource`来解析出具体的SQL语句。

我们从代码清单`3.20` 的70行`SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);`继续向下。

```java
//id:3.23
//package org.apache.ibatis.scripting.xmltags;
//XMLLanguageDriver
public SqlSource createSqlSource(
    Configuration configuration,
    XNode script,
    Class<?> parameterType) {
    
    XMLScriptBuilder builder 
        = new XMLScriptBuilder(configuration, script, parameterType);
    
    return builder.parseScriptNode();
}
```

这个方法，只是通过调用`XMLScriptBuilder`的`parseScriptNode()`来实现生成`SqlSource`的具体逻辑而已，因此我们继续向下

```java
//id:3.24
//package org.apache.ibatis.scripting.xmltags;
//XMLScriptBuilder
public SqlSource parseScriptNode() {
    //将context这个XNODE的所有子孙节点解析成一个MixedSqlNode（也就是一个SqlNode节点的列表）
    //并且设置isDynamic标志位，来表示这个sqlSource是否需要是动态的
    MixedSqlNode rootSqlNode = parseDynamicTags(context);
    
    SqlSource sqlSource;
    
    //根据是否是动态的创建不同的SqlSource实例
    if (isDynamic) {
      	sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    } else {
      	sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    }
    
    return sqlSource;
}
```

这里首先解释一下什么是动态SQL什么是静态SQL，动态SQL指的包含`${}`占位符或者`<if>`、`<where>`等动态语句节点的SQL。注意：只包含`#{}`并不算动态SQL。



在继续分析主要逻辑之前，我们先看看`MixSqlNode`是什么。对于每个`XNODE`片段，在经过解析后都会变成一个`SqlNode`节点，比如`TEXT`节点将被解析为一个`StaticTextSqlNode`，而`<if>`节点将被解析为一个`IfSqlNode`。比较特殊的是`MixSqlNode`，它存储一个`SqlNode`类型的列表。类图如下面两张图。

![](https://i.loli.net/2020/03/24/RwCIX1W248Hht6g.png)



![](https://i.loli.net/2020/03/24/oAYu5OQvDWMeGUm.png)



在大致了解了`SqlNode`之后，我们从代码清单`3.24`的第7行继续向下，看看`<select|insert|delete|update>`这个`XNode`是怎么被解析为一个`MixedSqlNode`的。下面源码的逻辑如下

1. 遍历`<select|insert|delete|update>`节点的所有子节点
2. 如果子节点是TEXT类型的，则根据是动态还是静态，解析`TextSqlNode`或者`StaticTextSqlNode`，并将解析结果放入`contents`列表
3. 如果子节点是ELEMENT类型的，那么根据标签名称来选取合适的`NodeHandler`解析，解析结果也会被放入`contents`列表
4. 最后通过第2步和第3步得到的`contents`列表，生成一个`MixedSqlNode`，并作为`SqlNode`树的根节点返回

```java
//id:3.25
//package org.apache.ibatis.scripting.xmltags;
//XMLScriptBuilder
protected MixedSqlNode parseDynamicTags(XNode node) {
    
    //一个SqlNode类型的列表，用来存储所有被解析成SqlNode的XNode
    List<SqlNode> contents = new ArrayList<>();
    
    //获取<select|insert|update|delete>节点的各个子SQL节点
    NodeList children = node.getNode().getChildNodes();
    
    //遍历所有子节点
    for (int i = 0; i < children.getLength(); i++) {
        XNode child = node.newXNode(children.item(i));
        
        //1.如果子节点是TEXT类型的
        if (child.getNode().getNodeType() == Node.CDATA_SECTION_NODE 
            || child.getNode().getNodeType() == Node.TEXT_NODE) {
            
            //1-1.获取节点中的具体SQL语句
            String data = child.getStringBody("");
            
            //1-2.通过data来创建一个TextSqlNode节点
            TextSqlNode textSqlNode = new TextSqlNode(data);
            
            //1-3.如果动态，则添加到第2行定义的contents中
            if (textSqlNode.isDynamic()) {
                contents.add(textSqlNode);
                isDynamic = true;
            
            //1-4.如果是静态的，则创建一个StaticTextSqlNode，并放入到contents中    
            } else {
                contents.add(new StaticTextSqlNode(data));
            }
            
        //2.如果子节点是ELEMENT类型的，也就是一<xxx>的节点    
        } else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) { // issue #628
            
            //2-1.获取标签的名称，例如:trim、where、if等
            String nodeName = child.getNode().getNodeName();
            
           	//2-2.根据名称获取不同的节点处理器
            NodeHandler handler = nodeHandlerMap.get(nodeName);
            
            //2-3.处理没有获取到处理器的情况
            if (handler == null) {
                throw new BuilderException("Unknown element <" 
                                           + nodeName 
                                           + "> in SQL statement.");
            }
            
            //2-4.调用处理器处理节点
            handler.handleNode(child, contents);
            
            //2-5.设置为动态
            isDynamic = true;
        }
    }
    
    //它是一个组合型节点，它会按顺序存储一个节点的列表
    return new MixedSqlNode(contents);

}
```



然后我们从上面代码清单的53行`handler.handleNode(child, contents);`向下，以一个`If`类型的`NodeHandler`为例，看看`ELEMENT`节点是怎么被解析的。

```java
//id:3.26
//package org.apache.ibatis.scripting.xmltags;
//XMLScriptBuilder.IfHandler
private class IfHandler implements NodeHandler {
    public IfHandler() {
      // Prevent Synthetic Access
    }

    @Override
    public void handleNode(XNode nodeToHandle, List<SqlNode> targetContents) {
        //对它的子节点再次调用parseDynamicTags，来生成一个MixSqlNode(相当于子节点列表)
      	MixedSqlNode mixedSqlNode = parseDynamicTags(nodeToHandle);
      	
        //从<if>节点(XNODE)上获取test属性
        String test = nodeToHandle.getStringAttribute("test");
      	
        //创建一个IF类型的SQLNODE节点
        IfSqlNode ifSqlNode = new IfSqlNode(mixedSqlNode, test);
      	
        //将这个节点添加到List中，也就是这个节点父节点的子节点列表
        targetContents.add(ifSqlNode);
    }
}
```



简单来说，就是对于`<select|create|insert|delete>`节点来说，它们的所有子节点内容将被解析为一棵节点类型为`SqlNode`的树，例如下图：这棵树储存在`MappedStatement.SqlSource.rootSqlNode`中，当运行时，用户调用传入具体参数，MyBatis就可以通过这棵树来生成具体的SQL语句了。至此，我们详细了解了SqlSource的生成过程，以及SqlSource的某些内部存储方式。



![](https://i.loli.net/2020/03/24/xW2ROFrvbgfpXAl.png)

##### 3.2.4.4 构建`MappedStatement`

接着，我们从代码清单`3.20`的93行`builderAssistant.addMappedStatement(xxx)`向下，看一下存储SQL语句节点解析结果的`MappedStatement`是如何构建的。

```java
//id:3.27
//package org.apache.ibatis.scripting.xmltags;
//XMLScriptBuilder.IfHandler
public MappedStatement addMappedStatement(
    String id,
    SqlSource sqlSource,
    StatementType statementType,
    SqlCommandType sqlCommandType,
    Integer fetchSize,
    Integer timeout,
    String parameterMap,
    Class<?> parameterType,
    String resultMap,
    Class<?> resultType,
    ResultSetType resultSetType,
    boolean flushCache,
    boolean useCache,
    boolean resultOrdered,
    KeyGenerator keyGenerator,
    String keyProperty,
    String keyColumn,
    String databaseId,
    LanguageDriver lang,
    String resultSets) {

    //处理有没有找到的缓存引用的情况
    if (unresolvedCacheRef) {
        throw new IncompleteElementException("Cache-ref not yet resolved");
    }

    //给id加上命名空间做前缀，以保证id的唯一性
    id = applyCurrentNamespace(id, false);
    
    //这个Sql语句是否为select语句
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    //使用建造者，并传入很多参数
    MappedStatement.Builder statementBuilder 
        = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
        .resource(resource)
        .fetchSize(fetchSize)
        .timeout(timeout)
        .statementType(statementType)
        .keyGenerator(keyGenerator)
        .keyProperty(keyProperty)
        .keyColumn(keyColumn)
        .databaseId(databaseId)
        .lang(lang)
        .resultOrdered(resultOrdered)
        .resultSets(resultSets)
        .resultMaps(getStatementResultMaps(resultMap, resultType, id))
        .resultSetType(resultSetType)
        .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
        .useCache(valueOrDefault(useCache, isSelect))
        .cache(currentCache);

    ParameterMap statementParameterMap 
        = getStatementParameterMap(parameterMap, parameterType, id);
  
    if (statementParameterMap != null) {
        statementBuilder.parameterMap(statementParameterMap);
    }

    //建造
    MappedStatement statement = statementBuilder.build();
    
    //将建造好的MappedStatement存储到Configuration中
    configuration.addMappedStatement(statement);
    return statement;
}
```



##### 3.2.4.5 总结

下面总结下本大节的内容。本节主要完成`<select|insert|delete|update>`节点的解析工作。每一个这类型节点通过解析后都会生成一个`MappedStatement`实例，储存具体的信息。对于它的`<inculde>`和`<selectKey>`子节点，将被解析替换为正常的SQL节点。然后在完成了替换后`<inculde>`和`<selectKey>`子节点将被从`XNode`树中删除。这之后，会解析这个干净的`XNode`树，每个具体的SQL语句节点将被转义并存储到可以一颗`SqlNode`类型的树中，在运行时，我们通过解析这棵树将获取具体的SQL语句。然后，我们把`MappedStatement`节点存储到`Configuration`中。一个`<select|insert|delete|update>`节点的解析工作就完成了。

### 3.3 Mapper接口绑定过程



当我们完成 了`<mapper>`文件的解析后，还需要通过绑定，将`<mapper>`文件中的每个SQL语句节点与java代码中对应`mapper`接口的对应方法绑定起来，存放到`Configuration.MapperRegistry`的`Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();`中，它也一个`Class`对象为键，以`MapperProxyFactory`为值，这个工厂可以通过反射为给类型的`mapper`接口生成实例。



这部分也不展开解释了，假如可以看懂第4章的sql执行过程，这个绑定过程也不在话下。



下面的代码是从代码清单`3.1`的17行`bindMapperForNamespace();`向下，完成具体的绑定过程。

```java
//id:3.28
//package org.apache.ibatis.builder.xml;
//XMLMapperBuilder
private void bindMapperForNamespace() {
    //获取这个<mapper>的命名空间
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
      Class<?> boundType = null;
        try {
            //通过命名空间找到对应的java类的Class对象
            boundType = Resources.classForName(namespace);
        } catch (ClassNotFoundException e) {
            //ignore, bound type is not required
        }
        if (boundType != null) {
            //如果这个java类还没有被解析过
            if (!configuration.hasMapper(boundType)) {
                // Spring may not know the real resource name so we set a flag
                // to prevent loading again this resource from the mapper interface
                // look at MapperAnnotationBuilder#loadXmlResource
                
                //将这个类加入已解析列表
                configuration.addLoadedResource("namespace:" + namespace);
                
                //将解析好的
                configuration.addMapper(boundType);
            }
        }
    }
}
```



接下来，我们从上述代码清单的26行`configuration.addMapper(boundType);`向下

```java
//id:3.29
//package org.apache.ibatis.binding;
//MapperRegistry
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
        if (hasMapper(type)) {
            throw new BindingException("Type " 
                                       + type 
                                       + " is already known to the MapperRegistry.");
        }
        
        boolean loadCompleted = false;
        
        try {
            knownMappers.put(type, new MapperProxyFactory<>(type));
            
            //下面是用来处理注解的
            // It's important that the type is added before the parser is run
            // otherwise the binding may automatically be attempted by the
            // mapper parser. If the type is already known, it won't try.
            MapperAnnotationBuilder parser 
                = new MapperAnnotationBuilder(config, type);
            parser.parse();
            loadCompleted = true;
        } finally {
            if (!loadCompleted) {
                knownMappers.remove(type);
            }
        }
    }
}
```



