---
title: '[MyBatis源码][4][SQL执行流程]'
date: 2020-03-27 22:21:01
tags:
    - MyBatis
categories:
    - MyBatis
---
## 第四章 SQL执行流程



通过前两章对配置文件和映射文件的解析，现在MyBatis已经进入了就绪状态，就等着使用者发号施令了。本章将对MyBatis执行SQL的过程进行详尽的分析，将包括如下技术点的1、2、6。剩余的技术点将在下面的章节展开。

1. 为`mapper`接口生成实现类
2. 根据配置信息生成SQL，并将运行时参数设置到SQL中
3. 一、二级缓存的实现
4. 插件机制
5. 数据库连接的获取与管理
6. 查询结果的处理，以及延迟加载



### 4.1 SQL解析入口

在使用MyBatis操作数据库的时候，我们通常会先调用`SqlSession`接口的`getMapper`方法为我们的`Mapper`接口生成实现类（代码清单`4.1`第10行）。然后就可以通过`Mapper`进行数据库操作（代码清单`4.1`第14行）



```java
//id:4.1
public class MyApp {
    public static void main(String[] args) throws IOException {
        Logger logger = Logger.getLogger(MyApp.class);
        InputStream inputStream = Resources.getResourceAsStream("mybatis.xml");
        SqlSessionFactory sqlSessionFactory 
            = new SqlSessionFactoryBuilder().build(inputStream);
        SqlSession sqlSession = sqlSessionFactory.openSession();
        
        //通过getMapper返回Mapper接口的实现类
        ProductDao productDao = sqlSession.getMapper(ProductDao.class);
        
        //通过Mapper进行数据库操作
        Product product= productDao.getProduct(12);
        
        logger.info(product);
    }
}
```

#### 4.1.1 为Mapper接口创建代理对象



我们首先跟随代码清单`4.1`第10行的调用栈向下，看看`getMapper`的具体实现方式



```java
//id:4.2
//package org.apache.ibatis.session.defaults;
//DefaultSqlSession
@Override
public <T> T getMapper(Class<T> type) {
	return configuration.getMapper(type, this);
}

//package org.apache.ibatis.session;
//Configuration
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
	return mapperRegistry.getMapper(type, sqlSession);
}


//package org.apache.ibatis.binding;
//MapperRegistry
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    
    //从knownMappers映射中，通过Class对象取得对应MapperProxyFactory(映射代理工厂)
    final MapperProxyFactory<T> mapperProxyFactory 
        = (MapperProxyFactory<T>) knownMappers.get(type);
    
    if (mapperProxyFactory == null) {
        throw new BindingException("Type " 
                                 + type 
                                 + " is not known to the MapperRegistry.");
    }
    
    try {
        //通过工厂，生成具体的Mapper
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}
```

上面代码清单的三层调用都非常简单，最后`Mapper`的创建，还是第32行的`return mapperProxyFactory.newInstance(sqlSession);`负责的。我们继续展开这个方法

```java
//id:4.3
//package org.apache.ibatis.binding;
//MapperProxyFactory<T>

public T newInstance(SqlSession sqlSession) {
    
    //传入三个参数
    //1. 会话(SqlSession sqlSession)；
    //2. 具体接口的Class对象(Class<T> mapperInterface)例如ProductMapper.class；
    //3. 缓存
    //来创建一个映射代理的实例，当调用代理对象的方法时，实际上都是调用它的`invoke`方法
    final MapperProxy<T> mapperProxy 
        = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    //调用重载方法
    return newInstance(mapperProxy);
}


protected T newInstance(MapperProxy<T> mapperProxy) {
    //通过Java的反射生成一个指定接口(mapperInterface)的代理对象
    return (T) Proxy.newProxyInstance(
        mapperInterface.getClassLoader(),
        new Class[] { mapperInterface },
        mapperProxy);
}
```



这里说一下`Proxy.newProxyInstance()`方法介绍。Proxy类的newInstance()方法有三个参数：
1. `ClassLoader loader`：它是类加载器类型，你不用去理睬它，你只需要知道怎么可以获得它就可以了：`MyInterface.class.getClassLoader()`就可以获取到`ClassLoader`对象，没错，只要你有一个`Class`对象就可以获取到`ClassLoader`对象；
2. `Class[] interfaces`：指定`newProxyInstance()`方法返回的对象要实现哪些接口，没错，可以指定多个接口，例如上面例子只我们只指定了一个接口：`Class[] cs = {MyInterface.class};`
3. `InvocationHandler handler`：它是最重要的一个参数！它是一个接口！它的名字叫调用处理器！无论你调用代理对象的什么方法，它都是在调用`InvocationHandler`的`invoke()`方法。



#### 4.1.2 执行代理逻辑

通过上一小节的分析，我们知道，实际调用代理对象的方法时，执行逻辑由`MapperProxy`的`invoke()`负责。继续分析这个`invoke()`方法

```java
//id:4.4
//package org.apache.ibatis.binding;
//MapperProxy<T>
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        } else if (isDefaultMethod(method)) {
            return invokeDefaultMethod(proxy, method, args);
        }
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
    
    //从缓存中寻找MapperMethod的对象，若未找到，则创建一个，并存入缓存中
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    
    //调用`execute`来执行SQL
    return mapperMethod.execute(sqlSession, args);
}
```

其实这个方法的主逻辑也很简单，首先从缓存中查询`MapperMethod`，如果找不到就创建一个并放入缓存。然后调用这个`MapperMethod`的`execute`方法执行实际的逻辑。



在4.1.2.1小节，我们首先会从代码清单`4.4`的16行展开，看看`MapperMethod`这个对象都存储了哪些数据，有什么功能。然后，在4.1.2.2小节，我们将从代码清单`4.4`的19行向下，详细分析`mapperMethod.execute()`方法



##### 4.1.2.1 创建`MappedMethod`对象

代码清单`4.4`的`16`行查找了一个缓存，这个缓存结构其实是`Map<Method, MapperMethod> methodCache;`，键是一个`Method`，而值为`MapperMethod`（没错，它又出现了，我们后面会分析它）。下面我们展开这个方法看看。

```java
//id:4.5
//package org.apache.ibatis.binding;
//MapperProxy<T>
private MapperMethod cachedMapperMethod(Method method) {
    return methodCache.computeIfAbsent(method, k -> new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
}
```

这个方法非常简单，其实调用的是java类库的`Map.computeIfAbsent`。`Map.computeIfAbsent`首先判断缓存`Map`中是否存在指定`key`的值，如果不存在，会自动调用`mappingFunction(key)`计算`key`的`value`，然后将`key = value`放入到缓存`Map`。



那接下来，我们好好看看前面一直提到的`MapperMethod`都存储哪些信息吧。它主要就只有两个成员变量：

1. `SqlCommand`主要存储和SQL有关的各种信息。通过`SqlCommand`我们可以找到对应的`MappedStatement`（详见第三章，每个`MappedStatement`都是对每个SQL语句节点解析的结果）。详见代码清单`4.7`
2. `MethodSignature`就是方法签名，它主要存储和目标方法相关的信息。包括返回值、参数列表这些。详见代码清单`4.8`

```java
//id:4.6
//package org.apache.ibatis.binding;
//MapperMethod
public class MapperMethod {

  	private final SqlCommand command;
  	private final MethodSignature method;

  	public MapperMethod(Class<?> mapperInterface, 
                        Method method,
                        Configuration config) {
    	
        this.command = new SqlCommand(config, mapperInterface, method);
    	this.method = new MethodSignature(config, mapperInterface, method);
  	}
}
```

`SqlCommand`对象如下：

```java
//id:4.7
//package org.apache.ibatis.binding;
//MapperMethod
public static class SqlCommand {

    private final String name;
    private final SqlCommandType type;

    public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
        
        final String methodName = method.getName();
        final Class<?> declaringClass = method.getDeclaringClass();
        MappedStatement ms 
            = resolveMappedStatement(mapperInterface, methodName, declaringClass,
          configuration);
        
        if (ms == null) {
            if (method.getAnnotation(Flush.class) != null) {
                name = null;
                type = SqlCommandType.FLUSH;
            } else {
                throw new BindingException("Invalid bound statement (not found): "
                + mapperInterface.getName() + "." + methodName);
            }
        
        } else {
        
            name = ms.getId();
            
            type = ms.getSqlCommandType();
            if (type == SqlCommandType.UNKNOWN) {
                throw new BindingException("Unknown execution method for: " + name);
            }
        }
    }
}
```

`MethodSignature`对象如下：

```java
//id:4.8
//package org.apache.ibatis.binding;
//MapperMethod
public static class MethodSignature {

    private final boolean returnsMany;
    
    private final boolean returnsMap;
    
    private final boolean returnsVoid;
    
    private final boolean returnsCursor;
    
    private final boolean returnsOptional;
    
    private final Class<?> returnType;
    
    private final String mapKey;
    
    private final Integer resultHandlerIndex;
    
    private final Integer rowBoundsIndex;
    
    private final ParamNameResolver paramNameResolver;

    public MethodSignature(Configuration configuration,
                           Class<?> mapperInterface,
                           Method method) {
        
        //获取给定Class对象和给定Method对象对应的方法返回值类型
        Type resolvedReturnType 
            = TypeParameterResolver.resolveReturnType(method, mapperInterface);
        
        //注：Class是Type的实现之一，Type是接口
        //如果Type是Class类型的
        if (resolvedReturnType instanceof Class<?>) {
            this.returnType = (Class<?>) resolvedReturnType;
        //如果Type是ParameterizedType类型的。
        } else if (resolvedReturnType instanceof ParameterizedType) {
            this.returnType 
                = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
        } else {
            this.returnType = method.getReturnType();
        }
        //总之从28-41行的代码主要是为了获取方法的returnType
        
        //判断是否返回为空
        this.returnsVoid = void.class.equals(this.returnType);
        
        //判断是否返回集合或数组
        this.returnsMany 
            = configuration.getObjectFactory().isCollection(this.returnType) 
            || this.returnType.isArray();
        
        //判断是否返回游标
        this.returnsCursor = Cursor.class.equals(this.returnType);
        
        //判断是否返回Optional
        this.returnsOptional = Optional.class.equals(this.returnType);
        
        
        this.mapKey = getMapKey(method);
        
        //判断是否返回Map
        this.returnsMap = this.mapKey != null;
        
        //找到RowBounds参数在Method方法的参数表中的位置
        this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
        
        //找到ResultHandler参数在Method方法的参数表中的位置
        this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
        
        //创建一个参数名称解析器，用来解析和存储参数列表
        this.paramNameResolver = new ParamNameResolver(configuration, method);
    }
}
```



然后我们从代码清单`4.8`的74行向下，看看`ParamNameResolver`具体是怎么进行参数解析的

```java
//id:4.9
//package org.apache.ibatis.reflection;
//ParamNameResolver
private final SortedMap<Integer, String> names;

private boolean hasParamAnnotation;

public ParamNameResolver(Configuration config, Method method) {
    //获取Method方法的参数类型数组
    final Class<?>[] paramTypes = method.getParameterTypes();
    //获取每个参数的每个注解
    final Annotation[][] paramAnnotations = method.getParameterAnnotations();
    //
    final SortedMap<Integer, String> map = new TreeMap<>();
    //参数的数量
    int paramCount = paramAnnotations.length;
    // get names from @Param annotations
    //遍历每个参数
    for (int paramIndex = 0; paramIndex < paramCount; paramIndex++) {
        
        //如果是特别参数（RowBounds类型或者ResultHandler类型）就跳过
        if (isSpecialParameter(paramTypes[paramIndex])) {
            // skip special parameters
            continue;
        }
        
        String name = null;
        
        //遍历这个参数的所有注解，寻找@Param注解
        for (Annotation annotation : paramAnnotations[paramIndex]) {
            
            //如果注解是@Param类型的
            if (annotation instanceof Param) {
                
                //设置有参数注解标记为True
                hasParamAnnotation = true;
                
                //从注解中获取值：例如从@Param("author")中获取author
                name = ((Param) annotation).value();
                //跳出
                break;
            }
        }
        
        //如果没有找到任何一个@Param
        if (name == null) {
            // @Param was not specified.
            //如果设置了使用实际参数名，则将名称设置为实际参数名
            if (config.isUseActualParamName()) {
                name = getActualParamName(method, paramIndex);
            }
            //如果没有设置，且没有@Param注解，那么就用参数的index(一个数字)作为名称
            if (name == null) {
                // use the parameter index as the name ("0", "1", ...)
                // gcode issue #71
                name = String.valueOf(map.size());
            }
        }
        //按照参数索引为k，解析到的参数名称为v的形式放入映射中
        map.put(paramIndex, name);
    }
    //使用map生成names，并冻结names
    names = Collections.unmodifiableSortedMap(map);
}
```

这个方法其实就是解析方法的每个参数，如果使用了`@Param`，就设置这个注解的值为参数名，否则就用参数的index(一个数字)作为名称。



下面我们举一个简单的例子。看看`select`方法的参数列表将被怎么解析

```java
//id:4.10
public interface ArticleMapper {
public void select(@Param("id") Integer id,
@Param("author") String author, RowBounds rb, Article article) {}
}
```

![](https://i.loli.net/2020/03/24/7CR3U4qhFvjnwlf.png)



##### 4.1.2.2 执行`execute`方法

书接代码清单`4.4`的19行`return mapperMethod.execute(sqlSession, args);`，我们来看看Mapper代理是如何执行具体逻辑的吧。其实这个方法主要根据不同的SQL类型，分派不同的逻辑，然后调用`SqlSession`中的具体方法

```java
//id:4.11
//package org.apache.ibatis.binding;
//MapperMethod
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    //根据MapperMethod.SqlCommand中存储的commandType来分派不同的逻辑
    switch (command.getType()) {
        //处理commandType为INSERT的情况
        case INSERT: {
            //将传入方法的参数转化为param(一会儿细看)
            Object param = method.convertArgsToSqlCommandParam(args);
            //实际调用sqlSession的insert
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
        }
        
        //略    
        case UPDATE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }
        
        //略  
        case DELETE: {
            Object param = method.convertArgsToSqlCommandParam(args);
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
        }
        
        //处理commandType为SELECT的情况
        case SELECT:
            //处理传入参数带ResultHandler的情况
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
            //处理需要返回List或者数组的情况
            } else if (method.returnsMany()) {
                result = executeForMany(sqlSession, args);
            //处理需要返回Map的情况
            } else if (method.returnsMap()) {
                result = executeForMap(sqlSession, args);
            //处理需要返回Cursor(游标)的情况
            } else if (method.returnsCursor()) {
                result = executeForCursor(sqlSession, args);
            //处理返回单个对象的情况
            } else {
                
                Object param = method.convertArgsToSqlCommandParam(args);
                result = sqlSession.selectOne(command.getName(), param);
                
                if (method.returnsOptional() 
                    && (result == null 
                        || !method.getReturnType().equals(result.getClass()))) {
                    
                    result = Optional.ofNullable(result);
                
                }
            }
            break;
        
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
            
        default:
            throw new BindingException("Unknown execution method for: " 
                                       + command.getName());
        }

    if (result == null 
        && method.getReturnType().isPrimitive() 
        && !method.returnsVoid()) {
        
        throw new BindingException("Mapper method '" 
            + command.getName()
            + " attempted to return null from a method with a primitive return type (" 
            + method.getReturnType() + ").");
    }
        
    return result;
}
```



代码清单`4.11`中，`rowCoundResule`经常出现，它其实就是根据方法签名里的返回值，为`Insert|Delete|Update`等操作返回不同的返回值，代码如下

```java
//id:4.12
//package org.apache.ibatis.binding;
//MapperMethod
//其实就是根据方法签名里的返回值，为Insert|Delete|update等操作返回不同的返回值
private Object rowCountResult(int rowCount) {
    final Object result;
    //返回值为void类型，则设置返回值为null
    if (method.returnsVoid()) {
        result = null;
    
    //返回值为Integer类型，则设置返回值为影响的行数
    } else if (Integer.class.equals(method.getReturnType()) 
               || Integer.TYPE.equals(method.getReturnType())) {
        result = rowCount;
    
    //返回值为Long类型，则设置返回值为影响的行数
    } else if (Long.class.equals(method.getReturnType()) 
               || Long.TYPE.equals(method.getReturnType())) {
        result = (long)rowCount;
    
    //返回值为Boolean类型，则设置返回值为影响的行数是否大于0    
    } else if (Boolean.class.equals(method.getReturnType()) 
               || Boolean.TYPE.equals(method.getReturnType())) {
        result = rowCount > 0;
    
    //否则抛出异常，Mapper不支持这种类型的返回
    } else {
        throw new BindingException("Mapper method '" 
                                   + command.getName() 
                                   + "' has an unsupported return type: " 
                                   + method.getReturnType());
    }
    return result;
}
```



### 4.2 查询语句的执行流程



在分析查询语句的执行流程之前，我们不妨先看一看。代码清单`4.11`的第49行`Object param = method.convertArgsToSqlCommandParam(args);`。回顾一下这一章前面介绍的`MappedMethod.MethodSigniture.ParamNameResolver`其中就存储了参数索引与参数名的映射：`private final SortedMap<Integer, String> names;`。所以我们现在要根据手头的参数索引与参数名映射，以及传入的参数列表，解析出一份参数名与参数值之间的映射。如代码片段`4.12`所示



```java
//id:4.12
//package org.apache.ibatis.binding;
//MapperMethod
//处理参数的过程
public Object convertArgsToSqlCommandParam(Object[] args) {
      return paramNameResolver.getNamedParams(args);
}

//package org.apache.ibatis.reflection;
//ParamNameResolver
public Object getNamedParams(Object[] args) {
    
    //查询之前的参数名称解析信息，看看有几个参数
    final int paramCount = names.size();

    //如果传入的args为空或者不需要参数，就直接返回一个空对象
    if (args == null || paramCount == 0) {
        return null;

    //如果有一个参数，就返回args中的第一个参数    
    } else if (!hasParamAnnotation && paramCount == 1) {  
        return args[names.firstKey()];
    
    //如果有多个参数，就创建一个Map<String, Object>
    //键为之前解析的MappedMethod中存储的参数名
    //值为传入的参数列表中的某个对应的值    
    } else {
        final Map<String, Object> param = new ParamMap<>();
        int i = 0;
        for (Map.Entry<Integer, String> entry : names.entrySet()) {
            
            //将这个参数的名称作为键，值作为值，存储到param映射中
            param.put(entry.getValue(), args[entry.getKey()]);
            
            // add generic param names (param1, param2, ...)
            //根据默认前缀和参数的索引，生成一个默认参数名
            final String genericParamName 
                = GENERIC_NAME_PREFIX + String.valueOf(i + 1);
            
            // ensure not to overwrite parameter named with @Param
            
            //把这个默认参数名作为键，值作为值，也放入到param映射中
            //针对同一个参数值，既可以通过指定的@Param名字访问，也可通过系统提供的默认参数名访问
            if (!names.containsValue(genericParamName)) {
                param.put(genericParamName, args[entry.getKey()]);
            }
            
            i++;
      }
      return param;
    }
}
```



#### 4.2.1 selectOne方法分析

接下来，我们仔细展开一种`selectOne`方法的调用栈，直到调用栈深至JDBC为止，看看MyBatis一共封了多长层，每一层分别完成什么逻辑。



首先是第一层：`SqlSession`层。它的上一层只会返回`List`，而这一层则对返回结果的类型进一步细分为`selectOne`、`selectList`、`selectCursor`等。下面我们从代码清单`4.11`，`SELECT`分支的第50行，`result = sqlSession.selectOne(command.getName(), param);`。向下，看看这个`selectOne`方法的内容。

```java
//id:4.13
//package org.apache.ibatis.session.defaults;
//DefaultSqlSession
public <T> T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
    List<T> list = this.selectList(statement, parameter);
    if (list.size() == 1) {
        return list.get(0);
    } else if (list.size() > 1) {
      	throw new TooManyResultsException("Expected one result (or null) to" 
                 + "be returned by selectOne(), but found: " 
                 + list.size());
    } else {
      return null;
    }
}
```

这个`selectOne`方法其实只是简单的调用`this.selectList(statement, parameter);`而已。然后取出返回的`List`中的唯一一个元素，返回。



下面我们继续往下走，再看看`selectList`都完成了哪些逻辑。

```java
//id:4.14
//package org.apache.ibatis.session.defaults;
//DefaultSqlSession
public <E> List<E> selectList(String statement, Object parameter) {
    //加上默认的分页RowBound
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
}

@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        //通过id来获取对应的MappedStatement
        MappedStatement ms 
            = configuration.getMappedStatement(statement);
        
        //调用Executor.query执行具体的查询逻辑并返回一个List<E>
        //从这个调用进入下一层
        return executor.query(ms,
                              wrapCollection(parameter),
                              rowBounds,
                              Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " 
                                             + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

第一层`SqlSession`层的逻辑到这里就结束啦，我们下面再来看看第二层`Executor`层都完成了哪些逻辑。



首先，展示`Executor`部分的类设计，它采用了装饰器模式

![](https://i.loli.net/2020/03/25/lh1yLBiJQ3vf4kG.png)

- 顶层接口是`Executor`，它定义了诸如查询、更新、提交、回滚等操作
- `BaseExector`是一个抽象类，它为它的4个子类实现了一些公共的处理逻辑，并且把特殊的处理逻辑设置为了抽象方法，交给子类实现
- `SimpleExecutor`是最常用的一个`Executor`
- 默认情况下，`Executor`的类型为`CachingExecutor`，该类是一个装饰器类，用于给目标`Executor`增加二级缓存。这个装饰器通常情况下修饰的是`SimpleExecutor`



接着我们打开`CachingExecutor`和`SimpleExecutor`的`query`方法来看看具体逻辑



```java
//id:4.15
//package org.apache.ibatis.executor;
//CachingExecutor

//MappedStatement: 它存储了Mapper文件中每个SQL语句节点的解析结果，通过它可以生成SQL语句

//parameterObject: 参数列表，如果只有一个参数就是它本身，有很多参数就是一个Map

//RowBounds: 特殊参数，用来处理分页

//ResultHandler: 特殊参数，用来处理返回结果（虽然一般都是通过MyBatis的自动映射来处理，但也可以自己定义结果处理器）

//CacheKey: 缓存相关（我也不懂）

//BoundSql boundSql，通过它可以获取这条Sql的各种信息。它其实是MappedStatement的成员变量
public <E> List<E> query(
    MappedStatement ms,
    Object parameterObject, 
    RowBounds rowBounds, 
    ResultHandler resultHandler, 
    CacheKey key, 
    BoundSql boundSql) throws SQLException {
    
    //获取这个MappedStatement对应的Cache
    Cache cache = ms.getCache();

    //如果Cache不是null
    if (cache != null) {
        
        flushCacheIfRequired(ms);

        if (ms.isUseCache() && resultHandler == null) {
            //确保没有输出参数
            ensureNoOutParams(ms, boundSql);
            @SuppressWarnings("unchecked")
        	//从事务缓存管理器中查询这次的sql是否有缓存结果
            List<E> list = (List<E>) tcm.getObject(cache, key);
            //没有就调用被装饰者的query，然后将查询结果放到缓存中
            if (list == null) {
                list = delegate.query(ms,
                                      parameterObject,
                                      rowBounds,
                                      resultHandler,
                                      key,
                                      boundSql);
                
                tcm.putObject(cache, key, list); // issue #578 and #116
            }
            //有则直接返回
            return list;
        }
    }
    
    //如果cache是null直接调用被装饰者的query
    return delegate.query(ms,
                          parameterObject,
                          rowBounds,
                          resultHandler,
                          key,
                          boundSql);
}
```

从上可以看到，`CacheExecutor`主要通过事务缓存管理器(`TransactionCacheManager`)，来完成缓存功能。对于这个类的分析将留在第6章。



那么接下来我们从代码清单`4.15`的55行`return delegate.query(ms,parameterObject,rowBounds,resultHandler,key,boundSql);`向下，看看被装饰者的`query`都执行了哪些逻辑

```java
//id:4.16
//package org.apache.ibatis.executor;
//BaseExecutor
@Override
public <E> List<E> query(MappedStatement ms,
                         Object parameter,
                         RowBounds rowBounds,
                         ResultHandler resultHandler,
                         CacheKey key,
                         BoundSql boundSql) throws SQLException {
    
    ErrorContext.instance().resource(ms.getResource())
        .activity("executing a query").object(ms.getId());
    
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }

    List<E> list;
    
    try {
        //一个用来控制延迟加载的标志
        queryStack++;
        
        //从一级缓存中查找
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        
        //存储过程相关处理逻辑，这里并不分析
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            
            //一级缓存未命中就从数据库中查询
            list = queryFromDatabase(ms,
                                     parameter,
                                     rowBounds,
                                     resultHandler,
                                     key,
                                     boundSql);
        }
    } finally {
        queryStack--;
    }

    if (queryStack == 0) {
        //从一级缓存中延迟加载嵌套查询结果
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        // issue #601
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // issue #482
            clearLocalCache();
        }
    }
    return list;
}
```

简单来说，这个方法会查询一级缓存，如果未命中，再向数据库进行查询。此外，这个方法还用于处理延迟加载的相关操作。



那我们继续从代码段`4.16`的38行`queryFromDatabase`向下，看看具体的数据库查询逻辑

```java
//id:4.17
//package org.apache.ibatis.executor;
//BaseExecutor
private <E> List<E> queryFromDatabase(MappedStatement ms,
                                      Object parameter,
                                      RowBounds rowBounds,
                                      ResultHandler resultHandler,
                                      CacheKey key,
                                      BoundSql boundSql) throws SQLException {
    
    List<E> list;
    
    //先往一级缓存中放入一个以CacheKey为键，占位符为值的映射
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    
    //执行具体的查询，这个`doQuery`是抽象方法，由BaseExecutor的4个子类实现
    try {
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        localCache.removeObject(key);
    }
    
    //向一级缓存中放入真正的查询结果
    localCache.putObject(key, list);
    
    //处理存储过程
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    
    return list;
}
```





接着，我们从代码清单`4.17`的第18行向下`list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);`。以`SimpleExecutor`为例，看看子类是具体怎么执行`doQuery`的

```java
//4.18
//package org.apache.ibatis.executor;
//SimpleExecutor
public <E> List<E> doQuery(MappedStatement ms,
                           Object parameter,
                           RowBounds rowBounds,
                           ResultHandler resultHandler,
                           BoundSql boundSql) throws SQLException {
    
    Statement stmt = null;
    
    try {
      	Configuration configuration = ms.getConfiguration();
      	//从此处进入下一层，创建一个Statement处理器  
      	StatementHandler handler 
          	= configuration.newStatementHandler(wrapper,
                                              ms,
                                              parameter,
                                              rowBounds,
                                              resultHandler,
                                              boundSql);
      	//创建这次查询的Statement  
      	stmt = prepareStatement(handler, ms.getStatementLog());
		
        //调用处理器执行查询
        return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```



从这个方法的第26行开始，我们正式从第二层(`Executor`)进入了第三层(`StatementHandler`)。这一层主要与JDBC交互，完成各种对数据库的操作。我们从`return handler.query(stmt, resultHandler);`向下，看看`handler`的`query`都干了什么。



```java
//4.19
//package org.apache.ibatis.executor.statement;
//PreparedStatementHandler
  @Override
public <E> List<E> query(Statement statement,
                         ResultHandler resultHandler) throws SQLException {
    
    String sql = boundSql.getSql();
    
    statement.execute(sql);
    
    return resultSetHandler.handleResultSets(statement);
}

```



这就是`selectOne`方法的全程，它分别经历了第一层`SqlSession`，第二层`Executor`和第三层`StatementHandler`。然后通过第三层，调用JDBC来执行具体的数据库操作。这就是主干设计，如下图。



但是除此之外还有很多重要的方法调用，例如代码清单`4.19`的`boundSql`。所以接下来的五个小节，我们讲补充这些枝干内容，以获取更深入的理解。

1. 4.2.2 将详细介绍如何将SQL语句完整的解析出来
2. 4.2.3 将介绍`StatementHandler`层的相关内容
3. 4.2.4 将介绍我们如何设置运行时参数给`Statement`
4. 4.2.5 将介绍对`#{}`的解析
5. 4.26 将介绍如何处理查询结果，将`ResultSet`解析成各种不同的类型



#### 4.2.2 获取BoundSql



先看看`BoundSql`的成员变量

```java
//4.20
//package org.apache.ibatis.mapping;
//BoundSql
public class BoundSql {

  private final String sql;
  private final List<ParameterMapping> parameterMappings;
  private final Object parameterObject;
  private final Map<String, Object> additionalParameters;
  private final MetaObject metaParameters;

}
```

| 变量名                  | 类型         | 用途                                                         |
| ----------------------- | ------------ | ------------------------------------------------------------ |
| `sql`                   | `String`     | 一个完整的SQL语句，包含问号`?`占位符                         |
| `parameterMappings`     | `List`       | 保存每个`#{}`占位符代表参数的基本信息，包括：`javaType`、`jdbcType`、名称等 |
| `parameterObject`       | `Object`     | 用户在运行时传入的具体参数值                                 |
| `additionnalParameters` | `Map`        | 附加参数，用来存储一些额外信息，比如`databaseId`等           |
| `metaParameters`        | `MetaObject` | `addtionalParameters`的元信息对象                            |



我们知道，要执行的Sql的最终解析信息都存储在了SqlSource中，那么我们看看如何获取这个`BoundSql`

```java
//4.21
//package org.apache.ibatis.mapping;
//MappedStatement
public BoundSql getBoundSql(Object parameterObject) {
    
    //直接调用SqlSource.getBoundSql()
    BoundSql boundSql 
        = sqlSource.getBoundSql(parameterObject);
    
    List<ParameterMapping> parameterMappings 
        = boundSql.getParameterMappings();
    
    //处理没有任何参数的情况
    if (parameterMappings == null || parameterMappings.isEmpty()) {
        boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
    }

    //处理关联的ResultMap，不重要
    // check for nested result maps in parameter mappings (issue #30)
    for (ParameterMapping pm : boundSql.getParameterMappings()) {
        String rmId = pm.getResultMapId();
        if (rmId != null) {
            ResultMap rm = configuration.getResultMap(rmId);
            if (rm != null) {
            hasNestedResultMaps |= rm.hasNestedResultMaps();
            }
        }
    }

    return boundSql;
}
```

其实就是简单的调用了`SqlSource.getBoundSql`，那我们继续向下





```java
//4.22
//package org.apache.ibatis.scripting.xmltags;
//DynamicSqlSource
@Override
public BoundSql getBoundSql(Object parameterObject) {
    
    //① 创建DynamicContext作为构建BoundSql所需的上下文，它依次添加每个SqlNode节点的解析结果
    DynamicContext context 
        = new DynamicContext(configuration, parameterObject);
    
    //② 解析SqlNode树，并将解析结果存储到`DynamicContext`中
    rootSqlNode.apply(context);
    
    
    SqlSourceBuilder sqlSourceParser 
        = new SqlSourceBuilder(configuration);
    
    Class<?> parameterType =
        parameterObject == null ? Object.class : parameterObject.getClass();
    
    //③ 构建StaticSqlSource，在此过程中将sql语句的占位符#{}替换为问号?
    //并且为每个占位符构建对应的`ParameterMapping`
    SqlSource sqlSource 
        = sqlSourceParser.parse(
        context.getSql(), parameterType, context.getBindings());
    
    //④ 调用StaticSqlSource的getBoundSql获取BoundSql
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    
    //⑤ 将DynamicContext的ContextMap中的内容拷贝给BoundSql
    context.getBindings().forEach(boundSql::setAdditionalParameter);
    
    return boundSql;
}
```



上述方法主要分为五个步骤

1. 创建`DynamicContext`作为构建`BoundSql`所需的上下文，它依次添加每个`SqlNode`节点的解析结果
2. 解析`SqlNode`树，并将解析结果存储到`DynamicContext`中
3. 构建一个`StaticSqlSource`，在此过程中将sql语句的占位符`#{}`替换为问号`?`，并且为每个占位符构建对应的`ParameterMapping`
4. 调用`StaticSqlSource`的`getBoundSql`获取`BoundSql`
5. 将`DynamicContext`的`ContextMap`中的内容拷贝给`BoundSql`

##### 4.2.2.1 创建`DynamicContext`

```java
//4.23
//package org.apache.ibatis.scripting.xmltags;
//DynamicContext
public class DynamicContext {

    public static final String PARAMETER_OBJECT_KEY = "_parameter";
    public static final String DATABASE_ID_KEY = "_databaseId";
  
    private final ContextMap bindings;
    private final StringJoiner sqlBuilder = new StringJoiner(" ");
    private int uniqueNumber = 0;
}
```

`sqIBuilder`变量用于存放SQL片段的解析结果， `bindings`则用于存储一些额外的信息，比如运行时参数和 `databased`等。

`DynamicContext`对外提供了两个接口，`appendSql`用来加入Sql片段，`getSql`用来获取SQL

##### 4.2.2.2 解析`SQL`片段

根据第三章的知识，我们都知道`SqlSource`中存储了一颗`SqlNode`树，当调用时，我们解析这颗`SqlNode`树就可以生成需要的sql语句，并存放到`DynamicContext`中。



`SqlNode`是一个接口，它只有一个`apply`方法

```java
//4.24
//package org.apache.ibatis.scripting.xmltags;
//SqlNode
public interface SqlNode {
  	boolean apply(DynamicContext context);
}
```

这个接口有很多实现类，用来处理不同类型的Sql节点，如下图。

![](https://i.loli.net/2020/03/26/rSZpMaACHTc5fO3.png)



下面举几个`SqlNode`的`apply`的例子



`MixSqlNode`中保存了一个SqlNode类型的`List`，它的`apply`就是调用`list`中所有`SqlNode`的`apply`方法

```java
//4.25
//package org.apache.ibatis.scripting.xmltags;
//MixedSqlNode
private final List<SqlNode> contents;
@Override
public boolean apply(DynamicContext context) {
    contents.forEach(node -> node.apply(context));
    return true;
}
```



`TextSqlNode`负责保存带有`${}`的SQL片段，因此它的`apply()`还需要把`${}`给替换为运行时传入的参数，使其静态化。

```java
//4.26
public boolean apply(DynamicContext context) {
    GenericTokenParser parser 
        = createParser(new BindingTokenParser(context, injectionFilter));
    context.appendSql(parser.parse(text));
    return true;
}
```

下面举个例子，假设有一个TextSql段

```sql
SELECT * FROM article WHERE author = ${authorId}
```

经过`TextSqlNode`的`apply()`之后，它会变成下面所示

```sql
SELECT * FROM article WHERE author = "bili"
```

具体的查找和替换过程这里不展开，这里要说明的是，对于`${}`占位符，在生成SQL语句的这一步就完成了替换，这是很危险的，容易产生SQL注入。

##### 4.2.2.3 解析`#{}`占位符

经过前面的解析，我们已经能从`DynamicContext`获取到完整的SQL语句了。但这并不意味着解析过程就结束了，因为当前的SQL语句中还有一种占位符没有处理，即`#{}`。



它的入口在代码清单`4.22`的23行`SqlSource sqlSource  = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());`。当这一步完成后不仅会，而且会



```java
//4.27
//package org.apache.ibatis.builder;
//SqlSourceBuilder
public SqlSource parse(String originalSql,
                       Class<?> parameterType,
                       Map<String, Object> additionalParameters) {
    
    ParameterMappingTokenHandler handler 
        = new ParameterMappingTokenHandler(configuration,
                                           parameterType,
                                           additionalParameters);
    
    GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
    
    String sql = parser.parse(originalSql);
    
    return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
  }
```



```java
//4.28
//package org.apache.ibatis.parsing;
//GenericTokenParser
public String parse(String text) {
    //如果要解析的内容是空，直接返回
    if (text == null || text.isEmpty()) {
      return "";
    }
    
    //根据openToken，在此次调用时是`#{`，找到它出现的为止
    // search open token
    int start = text.indexOf(openToken);
    
   	//如果位置为-1，也就是没有找到`#{`，直接返回
    if (start == -1) {
      return text;
    }
    
    //把传入的字符串转成src数组
    char[] src = text.toCharArray();
    
    //
    int offset = 0;
    
    final StringBuilder builder = new StringBuilder();
    StringBuilder expression = null;
    
    //循环，处理每一个`#{`
    while (start > -1) {
        
        //处理一种特殊情况
        if (start > 0 && src[start - 1] == '\\') {
        // this open token is escaped. remove the backslash and continue.
            builder.append(src, offset, start - offset - 1).append(openToken);
            offset = start + openToken.length();
        
        //处理正常的找到前缀的情况    
        } else {
            
            // found open token. let's search close token.
            //重置expression
            if (expression == null) {
                expression = new StringBuilder();
            } else {
                expression.setLength(0);
            }
            
            //先将前缀之前的部分全都加入到StringBuilder
            builder.append(src, offset, start - offset);
            offset = start + openToken.length();
            
            //寻找后缀`}`
            int end = text.indexOf(closeToken, offset);
            while (end > -1) {
                
                //处理一种特殊情况
                if (end > offset && src[end - 1] == '\\') {
                    // this close token is escaped. remove the backslash and continue.
                    expression.append(src,
                                      offset,
                                      end - offset - 1)
                        .append(closeToken);
                    offset = end + closeToken.length();
                    end = text.indexOf(closeToken, offset);
                
                    
                } else {
                    //否则将前后缀中间的内容，例如#{article}的article添加到expression中
                    expression.append(src, offset, end - offset);
                    offset = end + closeToken.length();
                    break;
                }
            }
            
            //处理没有找到后缀的情况
            if (end == -1) {
                // close token was not found.
                builder.append(src, start, src.length - start);
                offset = src.length;
            //
            } else {
                //使用handler.handleToken对expression进行处理
                //并将处理后的字符串也放入到builder中
                builder.append(handler.handleToken(expression.toString()));
                offset = end + closeToken.length();
            }
        }
        
        //继续寻找其他前缀`#{`
        start = text.indexOf(openToken, offset);
    }
    
    //将最后剩下的一部分字符，也放入builder
    if (offset < src.length) {
        builder.append(src, offset, src.length - offset);
    }
    
    //返回StringBuilder.toString()，把攒下来的SQL全部返回回去
    return builder.toString();
}
```



接下来我们看看83行的`handler.handleToken(expression.toString())`具体对找到的`expression`做了什么操作。

```java
//4.29
//package org.apache.ibatis.builder;
//SqlSourceBuilder.ParameterMappingTokenHandler
private List<ParameterMapping> parameterMappings = new ArrayList<>();

@Override
public String handleToken(String content) {
    //对content的内容进行解析，得到一个ParamterMapping，并放入到List中存放
    parameterMappings.add(buildParameterMapping(content));
    //直接返回"?"，意思是content将被"?"替代
    return "?";
}

//具体的解析过程，可以把类似#{age, javaType=int, jdbcType=NUMERIC, typeHandler=MyTypeHandler}这样的字符串，解析为一个PrameterMapping，具体的解析过程如下，不展开了
private ParameterMapping buildParameterMapping(String content) {
    
    Map<String, String> propertiesMap = parseParameterMapping(content);
    String property = propertiesMap.get("property");
    Class<?> propertyType;
    if (metaParameters.hasGetter(property)) { 
        // issue #448 get type from additional params
        propertyType = metaParameters.getGetterType(property);
    } else if (typeHandlerRegistry.hasTypeHandler(parameterType)) {
        propertyType = parameterType;
    } else if (JdbcType.CURSOR.name().equals(propertiesMap.get("jdbcType"))) {
        propertyType = java.sql.ResultSet.class;
    } else if (property == null || Map.class.isAssignableFrom(parameterType)) {
        propertyType = Object.class;
    } else {
        MetaClass metaClass 
            = MetaClass.forClass(parameterType, configuration.getReflectorFactory());
        if (metaClass.hasGetter(property)) {
            propertyType = metaClass.getGetterType(property);
        } else {
            propertyType = Object.class;
        }
    }

    ParameterMapping.Builder builder 
        = new ParameterMapping.Builder(configuration, property, propertyType);
    Class<?> javaType = propertyType;
    String typeHandlerAlias = null;
    for (Map.Entry<String, String> entry : propertiesMap.entrySet()) {
        String name = entry.getKey();
        String value = entry.getValue();
        if ("javaType".equals(name)) {
            javaType = resolveClass(value);
            builder.javaType(javaType);
        } else if ("jdbcType".equals(name)) {
            builder.jdbcType(resolveJdbcType(value));
        } else if ("mode".equals(name)) {
            builder.mode(resolveParameterMode(value));
        } else if ("numericScale".equals(name)) {
            builder.numericScale(Integer.valueOf(value));
        } else if ("resultMap".equals(name)) {
            builder.resultMapId(value);
        } else if ("typeHandler".equals(name)) {
            typeHandlerAlias = value;
        } else if ("jdbcTypeName".equals(name)) {
            builder.jdbcTypeName(value);
        } else if ("property".equals(name)) {
            // Do Nothing
        } else if ("expression".equals(name)) {
            throw new BuilderException("Expression based parameters yet");
        } else {
            throw new BuilderException("An invalid property '" 
                                       + name 
                                       + "' was found in mapping #{" 
                                       + content 
                                       + "}.  Valid properties are " 
                                       + PARAMETER_PROPERTIES);
        }
    }
    if (typeHandlerAlias != null) {
        builder.typeHandler(resolveTypeHandler(javaType, typeHandlerAlias));
    }
    return builder.build();
}
```



通过了这一步的解析，Sql语句放入了`DynamicContext`和`ParamterMappings`放入了`ParameterMappingTokenHandler`，然后我们通过这些信息构建一个StaticSqlSource并返回即可，如代码清单`4.27`的17行`return new StaticSqlSource(configuration, sql, handler.getParameterMappings());`所示



##### 4.2.2.4 总结

至此，我们获取到了`BoundSql`，里面包含的SQL语句和形参列表再后面的步骤将会起到作用。



#### 4.2.3 创建StatementHandler



在`MyBatis`的源码中，`StatementHandler`是一个非常核心接口。之所以说它核心，是因为从代码分层的角度来说，` StatementHandler`是 `MyBatis`源码的边界，再往下层就是`JDBC`层面的接口了。`StatementHandler`需要和JDBC层面的接口打交道。

它要做的事情有很多。

1. 在执行SQL之前，`StatementHandler`需要创建合适的 `Statement`对象，然后填充参数值到`Statement`对象中，最后通过 `Statement`对象执行SQL。
2. 待SQL执行完毕，还要去处理查询结果等

这些过程看似简单，但实现起来却很复杂。好在，这些过程对应的逻辑并不需要我们亲自实现。好了，其他的就不多说了。下面我们来看一下 `StatementHandler`的继承体系。

![](https://i.loli.net/2020/03/26/JP1NnTAm3l2dceO.png)

- `StatementHandler`是一个接口，它定义了各种操作：`prepare`、`batch`、`update`、`query`等。
- `BaseStatementHandler`是一个抽象类，它实现了接口的一部分方法。
- 它的三个子类，分别对应三种不同的`Statement`：`Statement`、`CallableStatement`、`PreparedStatement`。
- 而`RoutingStatementHandler`是一个装饰器，它会根据传入的`MappedStatement`中的`statementType`选用不同的`handler`进行处理。



接下来我们从代码清单`4.18`的第15行`StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);`，来看看`StatementHandler`的具体创建过程。

```java
//4.30
//package org.apache.ibatis.session;
//Configuration
public StatementHandler newStatementHandler(Executor executor,
                                            MappedStatement mappedStatement,
                                            Object parameterObject,
                                            RowBounds rowBounds,
                                            ResultHandler resultHandler,
                                            BoundSql boundSql) {
    //创建一个StatementHandler
    StatementHandler statementHandler 
        = new RoutingStatementHandler(executor,
                                      mappedStatement,
                                      parameterObject,
                                      rowBounds,
                                      resultHandler,
                                      boundSql);
    
    //为statementHandler添加插件
    statementHandler 
        = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    
    return statementHandler;
}
```



```java
//4.31
//RoutingStatementHandler
public class RoutingStatementHandler implements StatementHandler {
  //被装饰的类
  private final StatementHandler delegate;
  //构造器根据不同的MappedStatement.statementType来创建不同的被装饰的类的实例
  public RoutingStatementHandler(Executor executor,
                                 MappedStatement ms,
                                 Object parameter,
                                 RowBounds rowBounds,
                                 ResultHandler resultHandler,
                                 BoundSql boundSql) {

    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate 
            = new SimpleStatementHandler(executor, ms, parameter,
                                         rowBounds, resultHandler, boundSql);
        break;
            
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter,
                                                rowBounds, resultHandler, boundSql);
        break;
            
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter,
                                                rowBounds, resultHandler, boundSql);
        break;
            
      default:
        throw new ExecutorException("Unknown statement type: " 
                                    + ms.getStatementType());
    }
  }
  //这个类的其他方法都是委托给被装饰类执行，例如下面这个
  @Override
  public <E> List<E> query(Statement statement,
                           ResultHandler resultHandler) throws SQLException {
    return delegate.query(statement, resultHandler);
  }  
}
```

`RoutingStatementHandler`的构造方法会根据`MappedStatement`中的`statementType`变量创建不同的 `Statementhandler`实现类。默认情况下，`statementType`值为`PREPARED`。关于`StatementHandler`创建的过程就先分析到这， `StatementHandler`创建完成了，后续要做到事情是创建 `Statement`，以及将运行时参数和 `Statement`进行绑定。



#### 4.2.4 创建`Statement`并设置运行时参数



本章将从`4.18`的23行`stmt = prepareStatement(handler, ms.getStatementLog());`向下，看看如何创建 `Statement`，以及如何将运行时参数和 `Statement`进行绑定。

```java
//4.32
//SimpleExecutor
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    //创建Statement
    stmt = handler.prepare(connection, transaction.getTimeout());
    //绑定运行时参数
    handler.parameterize(stmt);
    return stmt;
}
```



```java
//4.33
//BaseStatementHandler
public Statement prepare(Connection connection, 
                         Integer transactionTimeout) throws SQLException {
    
    ErrorContext.instance().sql(boundSql.getSql());
    
    Statement statement = null;
    
    try {
        //其实就是根据不同的keyGenerate和ResultHandler情况，调用connection.prepareStatement
        statement = instantiateStatement(connection);
        
        setStatementTimeout(statement, transactionTimeout);
        
        setFetchSize(statement);
        
        return statement;
    
    } catch (SQLException e) {
        
        closeStatement(statement);
        throw e;
    } catch (Exception e) {
        
        closeStatement(statement);
        throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
}
```



```java
//4.34
//PreparedStatementHandler
public void parameterize(Statement statement) throws SQLException {
    parameterHandler.setParameters((PreparedStatement) statement);
}
```

xx

```java
//4.35
//DefaultParameterHandler
public void setParameters(PreparedStatement ps) {
    
    ErrorContext.instance().activity("setting parameters").
        object(mappedStatement.getParameterMap().getId());
    
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    
    if (parameterMappings != null) {
        //遍历参数列表中的全部参数
        for (int i = 0; i < parameterMappings.size(); i++) {
            
            ParameterMapping parameterMapping = parameterMappings.get(i);
            
            //不处理OUT类型的参数
            if (parameterMapping.getMode() != ParameterMode.OUT) {
                
                Object value;
                //取得当前要处理的参数名
                String propertyName = parameterMapping.getProperty();
                
                //通过各种方式获取参数名对应的参数值
                if (boundSql.hasAdditionalParameter(propertyName)) { 
                    // issue #448 ask first for additional params
                    value = boundSql.getAdditionalParameter(propertyName);
                
                } else if (parameterObject == null) {
                    value = null;
                
                } else if (typeHandlerRegistry.
                           hasTypeHandler(parameterObject.getClass())) {   
                    value = parameterObject;
                
                } else {
                    MetaObject metaObject 
                        = configuration.newMetaObject(parameterObject);
                    value = metaObject.getValue(propertyName);
                }
                
                //获取对应的类型处理器
                TypeHandler typeHandler = parameterMapping.getTypeHandler();
                
                //获取对应的JdbcType
                JdbcType jdbcType = parameterMapping.getJdbcType();
                
                //对Null类型的JdbcType进行特殊处理
                if (value == null && jdbcType == null) {
                    jdbcType = configuration.getJdbcTypeForNull();
                }
                
                //调用typeHandler.setParameter来完成参数的设置工作
                try {
                    typeHandler.setParameter(ps, i + 1, value, jdbcType);
                } catch (TypeException | SQLException e) {
                    throw new TypeException("Could not set parameters for mapping: " 
                                            + parameterMapping 
                                            + ". Cause: " + e, e);
                }
            }
        }
    }
}
```



#### 4.2.5 处理查询结果



MyBatis可以将查询结果，即结果集 `ResultSet`自动映射成实体类对象。这样使用者就无需再手动操作结果集，并将数据填充到实体类对象中。这可大大降低开发的工作量，提高工作效率。在 MyBatis中，结果集的处理工作由结果集处理器 `ResultSetHandler`执行。
`ResultSetHandler`是一个接口，它只有一个实现类 `DefaultResultSetHandler`。结果集的处理入口方法是 `handleResultSets`，下面来看一下该方法的实现。



代码清单`4.19`的第12行`return resultSetHandler.handleResultSets(statement);`负责对执行结果进行处理，它将返回一个`List<Object>`

```java
//4.36
//DefaultResultSetHandler
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance()
        .activity("handling results").object(mappedStatement.getId());

    //新建一个List来存储结果
    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    //获取第一个结果集
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    //ResultMap主要存放了这个Statement的返回值相关信息，包括SQL-Java映射等
    //里面有一个`ResultMapping列表
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    
    
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    
    //处理单个结果集
    while (rsw != null && resultMapCount > resultSetCount) {
        ResultMap resultMap = resultMaps.get(resultSetCount);
        //处理结果集
        handleResultSet(rsw, resultMap, multipleResults, null);
        //获取下一个结果集
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
    }

    //以下的处理与多结果集有关，这里直接跳过
    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
        while (rsw != null && resultSetCount < resultSets.length) {
            ResultMapping parentMapping 
                = nextResultMaps.get(resultSets[resultSetCount]);
            if (parentMapping != null) {
                String nestedResultMapId = parentMapping.getNestedResultMapId();
                ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
                handleResultSet(rsw, resultMap, null, parentMapping);
            }
            rsw = getNextResultSet(stmt);
            cleanUpAfterHandlingResultSet();
            resultSetCount++;
        }
    }

    return collapseSingleResultList(multipleResults);
}
```

下面从代码清单4.36的26行向下`handleResultSet(rsw, resultMap, multipleResults, null);`，看看单结果集的处理逻辑

```java
//4.37
//DefaultResultSetHandler
private void handleResultSet(ResultSetWrapper rsw,
                             ResultMap resultMap,
                             List<Object> multipleResults,
                             ResultMapping parentMapping) throws SQLException {
    try {
      //处理多结果集相关逻辑，跳过  
      if (parentMapping != null) {
            handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      //处理正常情况下，单结果集
      } else {
        //如果用户没有配置resultHandler，则创建一个默认的，来处理单行数据  
        if (resultHandler == null) {
            //创建一个默认的结果处理器
            DefaultResultHandler defaultResultHandler 
                = new DefaultResultHandler(objectFactory);
            //处理单行数据
            handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
            //将处理后的结果加入到结果列表
            multipleResults.add(defaultResultHandler.getResultList());
        //如果用户配置了resultHandler，则直接使用它来处理单行数据
        } else {
            //处理单行数据
            handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
        // issue #228 (close resultsets)
        closeResultSet(rsw.getResultSet());
    }
}
```



我们先看看前面几个代码清单中一直出现的`ResultSetWrapper`，它是一个存储数据的类，它持有一个JDBC的`ResultSet`的引用，并且还保存了诸如列名列表、类名列表、JDBC类型列表等等其他信息。

```java
//4.38
//ResultSetWrapper
public class ResultSetWrapper {

    private final ResultSet resultSet;
    private final TypeHandlerRegistry typeHandlerRegistry;
    private final List<String> columnNames = new ArrayList<>();
    private final List<String> classNames = new ArrayList<>();
    private final List<JdbcType> jdbcTypes = new ArrayList<>();
    private final Map<String, Map<Class<?>, TypeHandler<?>>> typeHandlerMap 
        = new HashMap<>();
    private final Map<String, List<String>> mappedColumnNamesMap = new HashMap<>();
    private final Map<String, List<String>> unMappedColumnNamesMap = new HashMap<>();
}
```



我们继续从19行`handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);`向下，看看如何处理单行数据。

```java
//4.39
//DefaultResultSetHandler
public void handleRowValues(ResultSetWrapper rsw,
                            ResultMap resultMap,
                            ResultHandler<?> resultHandler,
                            RowBounds rowBounds,
                            ResultMapping parentMapping) throws SQLException {
    //处理嵌套映射
    if (resultMap.hasNestedResultMaps()) {
      
        ensureNoRowBounds();
      
        checkResultHandler();
      
        handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler,
                                          rowBounds, parentMapping);
    //处理简单映射
    } else {
      handleRowValuesForSimpleResultMap(rsw,
                                        resultMap,
                                        resultHandler,
                                        rowBounds,
                                        parentMapping);
    }
}
```

这个方法只是通过两条分支来处理嵌套映射和简单映射而已。我们从19行向下，继续看看`handleRowValuesForSimpleResultMap`如何处理简单映射



```java
//4.40
//DefaultResultSetHandler
private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw,
                    ResultMap resultMap,
                    ResultHandler<?> resultHandler,
                    RowBounds rowBounds,
                	ResultMapping parentMapping)throws SQLException {
    
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
   	//从rsw中拿到resultSet
    ResultSet resultSet = rsw.getResultSet();
    //根据RowBounds定位到指定行记录
    skipRows(resultSet, rowBounds);
    //检测是否还有更多行的数据需要处理
    while (shouldProcessMoreRows(resultContext, rowBounds) 
           && !resultSet.isClosed() && resultSet.next()) {
        //鉴别器处理ResultMap
        ResultMap discriminatedResultMap 
            = resolveDiscriminatedResultMap(resultSet, resultMap, null);
        //从resultSet中获取结果
        Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
        //存储结果
        storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
}
```

上面方法的逻辑主要有

1. 根据`RowBounds`定位到指定行记录
2. 循环处理多行数据
3. 使用鉴别器处理`ResultMap`
4. 映射`ResultMap`得到映射结果`rowValue`
5. 存储结果

下面我们具体展开第1步和第4步



首先是从代码清单`4.40`的13行`skipRows(resultSet, rowBounds);`的第一步。

```java
//4.41
//DefaultResultSetHandler
private void skipRows(ResultSet rs, RowBounds rowBounds) throws SQLException {
    //检测rs的类型，不同的类型行数据定位方式是不同的
    if (rs.getType() != ResultSet.TYPE_FORWARD_ONLY) {
        if (rowBounds.getOffset() != RowBounds.NO_ROW_OFFSET) {
            rs.absolute(rowBounds.getOffset());
        }
    //如果不能采用上述方式    
    } else {
        //遍历结果集，通过一步一步的rs.next()来定位行数据，非常低效
        for (int i = 0; i < rowBounds.getOffset(); i++) {
            if (!rs.next()) {
                break;
            }
        }
    }
}
```

MyBatis默认提供了`RowBounds`用于分页，从上面的代码中可以看出，这并非是一个高效的分页方式。除了使用 `RowBounds`，还可以使用一些第三方分页插件进行分页。



然后是从代码清单`4.40`的21行`Object rowValue = getRowValue(rsw, discriminatedResultMap, null);`的第四步，映射`ResultMap`得到映射结果`rowValue`的具体逻辑。

```java
//4.42
//DefaultResultSetHandler
private Object getRowValue(ResultSetWrapper rsw,
                           ResultMap resultMap,
                           String columnPrefix) throws SQLException {
    
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    
    //1. 创建实体类对象，来装数据
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
    
    //如果创建成功，并且没有这个类型的TypeHandler
    if (rowValue != null 
        && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        
        final MetaObject metaObject = configuration.newMetaObject(rowValue);
        
        boolean foundValues = this.useConstructorMappings;
        
        //2. 检测是否采用自动映射
        if (shouldApplyAutomaticMappings(resultMap, false)) {
            //进行自动映射
            foundValues 
                = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) 
                || foundValues;
        }
        //3. 根据resultMap中配置的SQL-Java映射来进行映射
        foundValues 
            = applyPropertyMappings(rsw, resultMap, metaObject,
                                    lazyLoader,columnPrefix) 
            || foundValues;
        
        foundValues = lazyLoader.size() > 0 || foundValues;
        
        rowValue = foundValues 
            || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
}
```

上述代码主要分为如下逻辑

1. 创建实体类对象。4.2.5.1小节将展开分析。
2. 检测结果集是否需要自动映射，若需要则进行自动映射。4.2.5.2小节将展开分析
3. 按照`<resultMap>`中配置的映射关系进行映射。4.2.5.3小节将展开分析

##### 4.2.5.1 创建实体类对象

对于创建实体类对象，MyBatis的维护者写了很多逻辑，以保证能成功创建实体类对象。如果实在无法创建，则抛出异常。



下面从代码清单`4.42`的第10行`Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);`向下，看看如何创建实体类对象。

```java
//4.43
//DefaultResultSetHandler
private Object createResultObject(ResultSetWrapper rsw,
                                  ResultMap resultMap,
                                  ResultLoaderMap lazyLoader,
                                  String columnPrefix) throws SQLException {
    
    this.useConstructorMappings = false; // reset previous mapping result
    
    final List<Class<?>> constructorArgTypes = new ArrayList<>();
    
    final List<Object> constructorArgs = new ArrayList<>();
    
    //调用重载方法创建实体类对象
    Object resultObject 
        = createResultObject(rsw,
                             resultMap,
                             constructorArgTypes,
                             constructorArgs,
                             columnPrefix);
    //检测实体类是否有相应的类型处理器，如果没有则
    if (resultObject != null 
        && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        
        final List<ResultMapping> propertyMappings 
            = resultMap.getPropertyResultMappings();
        //遍历ResultMap中的每个ResultMapping
        for (ResultMapping propertyMapping : propertyMappings) {
            // issue gcode #109 && issue #149
            //如果有任何一个ResultMapping开启了延迟加载，则
            if (propertyMapping.getNestedQueryId() != null 
                && propertyMapping.isLazy()) {
                //为这个实体类创建一个代理对象，用来处理延迟加载
                resultObject 
                    = configuration.getProxyFactory()
                    .createProxy(resultObject, lazyLoader, configuration,
                                 objectFactory, constructorArgTypes, constructorArgs);
                break;
            }
        }
    }
    
    this.useConstructorMappings 
        = resultObject != null && !constructorArgTypes.isEmpty(); 
    
    return resultObject;
}
```

创建实体类对象的逻辑被封装在了`createResultObject`的重载方法中。在创建好实体类之后，还需要对`<resultMap>`中配置的映射信息进行检测。若发现有关联查询，且关联查询结果的加载方式是延迟加载，就为实体类生成代理类。



然后我们从代码清单`4.43`的15行向下，看看`createResultObject`的重载方法

```java
//4.44
//DefaultResultSetHandler
private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)
      throws SQLException {
    
    final Class<?> resultType = resultMap.getType();
    
    final MetaClass metaType = MetaClass.forClass(resultType, reflectorFactory);
    //获取<constructor>节点对应的ResultMapping
    final List<ResultMapping> constructorMappings 
        = resultMap.getConstructorResultMappings();
    //检测是否有与返回结果类型相同的TypeHandler，如果有则直接使用它来处理，并生成返回值对象
    if (hasTypeHandlerForResultObject(rsw, resultType)) {
        return createPrimitiveResultObject(rsw, resultMap, columnPrefix);
    //如果<resultMap>中定义了构造器，则使用构造器来创建一个对象
    } else if (!constructorMappings.isEmpty()) {
        return createParameterizedResultObject(rsw, resultType, constructorMappings,
                                               constructorArgTypes, constructorArgs,
                                               columnPrefix);
    //如果没有定义构造器，则直接使用默认的无参构造器来创建一个对象
    } else if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
        return objectFactory.create(resultType);
    //处理自动映射
    } else if (shouldApplyAutomaticMappings(resultMap, false)) {
        return createByConstructorSignature(rsw, resultType,
                                            constructorArgTypes, constructorArgs);
    //如果前面的每个判断都不成立，则抛出异常
    }else{
        throw new ExecutorException("Do not know how to create an instance of " 
                                    + resultType);
    }
}
```



`createResultObject`方法中包含了4 种创建实体类对象的方式。一般情况下，若无特殊要求，MyBatis 会通过`ObjectFactory` 调用默认构造方法创建实体类对象。`ObjectFactory` 是一个接口，大家可以实现这个接口，以按照自己的逻辑控制对象的创建过程。



##### 4.2.5.2 自动映射属性到实体类对象

在创建了实体类之后，我们需要把结果集中的属性映射到实体类中。这个过程分为两步

1. 通过自动映射，将一部分没有在`<ResultMap>`中出现的属性的值设置到实体类中
2. 将`<ResultMap>`中出现的属性的值设置到实体类中



本小节先分析第一步，下一小节分析第二步。



下面从代码清单`4.42`的23行`foundValues  = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix)  || foundValues;`向下，看看自动映射的逻辑

```java
////4.45
//DefaultResultSetHandler
private boolean applyAutomaticMappings(ResultSetWrapper rsw,
                                       ResultMap resultMap,
                                       MetaObject metaObject,
                                       String columnPrefix) throws SQLException {
    //获取UnMappedColumnAutoMapping列表
    List<UnMappedColumnAutoMapping> autoMapping 
        = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);
    
    boolean foundValues = false;
    
    //如果这个列表不是空的
    if (!autoMapping.isEmpty()) {
        //遍历每一个Mapping
        for (UnMappedColumnAutoMapping mapping : autoMapping) {
            
            //通过TypeHandler从结果集中获取结果
            final Object value 
                = mapping.typeHandler.getResult(rsw.getResultSet(), mapping.column);
            
            if (value != null) {
                foundValues = true;
            
            }
            
            if (value != null 
                || (configuration.isCallSettersOnNulls() && !mapping.primitive)) {
                //通过元信息对象设置value到实体类对象的指定字段上
                // gcode issue #377, call setter on nulls (value is not 'found')
                metaObject.setValue(mapping.property, value);
            }
        }
    }
    return foundValues;
}
```



我们接着看看上述代码清单中出现`UnMappedColumnAutoMapping`类中都存储了什么信息。它主要用来存储每一个未出现在`<resultMap>`中的属性。

```java
//4.46
//DefaultResultSetHandler.UnMappedColumnAutoMapping
private static class UnMappedColumnAutoMapping {
    private final String column;
    private final String property;
    private final TypeHandler<?> typeHandler;
    private final boolean primitive;

    public UnMappedColumnAutoMapping(String column, String property, TypeHandler<?> typeHandler, boolean primitive) {
      this.column = column;
      this.property = property;
      this.typeHandler = typeHandler;
      this.primitive = primitive;
    }
}

```



下面我们紧接代码清单`4.45`的第8行`List<UnMappedColumnAutoMapping> autoMapping = createAutomaticMappings(rsw, resultMap, metaObject, columnPrefix);`，看看这个未出现在`<resultMap>`中的属性的列表是如何构建出来的

```java
//4.47
//DefaultResultSetHandler
private List<UnMappedColumnAutoMapping> createAutomaticMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String columnPrefix) throws SQLException {
    
    final String mapKey = resultMap.getId() + ":" + columnPrefix;
    //从缓存中获取UnMappedColumnAutoMappings
    List<UnMappedColumnAutoMapping> autoMapping = autoMappingsCache.get(mapKey);
    //缓存中没有就创建
    if (autoMapping == null) {
        
        autoMapping = new ArrayList<>();
        
        //从ResultSetWrapper中获取未配置在`<resultMap>`中的列名
        final List<String> unmappedColumnNames 
            = rsw.getUnmappedColumnNames(resultMap, columnPrefix);
        
        //遍历每个列名
        for (String columnName : unmappedColumnNames) {
            String propertyName = columnName;
            
            //有前缀的话就去掉前缀
            if (columnPrefix != null && !columnPrefix.isEmpty()) {
                // When columnPrefix is specified,
                // ignore columns without the prefix.
                if (columnName.toUpperCase(Locale.ENGLISH).startsWith(columnPrefix)) {
                    propertyName = columnName.substring(columnPrefix.length());
                } else {
                    continue;
                }
            }
            
            //将下划线形式的列名转化为驼峰形式，例如:AUTHOR_NAME -> authorName
            final String property 
                = metaObject.findProperty(propertyName,
                                          configuration.isMapUnderscoreToCamelCase());
            
            //如果属性不为null，且实体类中有这个属性的setter
            if (property != null && metaObject.hasSetter(property)) {
            	//如果转码后发现其实它已经被映射过了，就直接跳出
                if (resultMap.getMappedProperties().contains(property)) {
                    continue;
                }
                
                //获取这个属性对应的类型
                final Class<?> propertyType = metaObject.getSetterType(property);
                
                //找一找有没有对应的TypeHandler
                if (typeHandlerRegistry
                    .hasTypeHandler(propertyType, rsw.getJdbcType(columnName))) {
                    
                    //获取TypeHandler
                    final TypeHandler<?> typeHandler 
                        = rsw.getTypeHandler(propertyType, columnName);
                    //将前面拿到的各种信息放到一个UpMappedColumnAutoMapping的列表中
                    autoMapping.add(
                        new UnMappedColumnAutoMapping(columnName, property,
                                                      typeHandler,
                                                      propertyType.isPrimitive()));
                //处理没有找到对应TypeHandler的情况
                } else {
                    configuration.getAutoMappingUnknownColumnBehavior()
                        .doAction(mappedStatement, columnName,
                                  property, propertyType);
                }
            //处理没有找到对应属性的情况，有三种处理方式
            //1. 抛出异常
            //2. 什么都不做
            //3. 仅打印日志    
            } else {
                configuration.getAutoMappingUnknownColumnBehavior()
                    .doAction(mappedStatement, columnName,
                              (property != null) ? property : propertyName, null);
            }

        }
        //写入缓存
        autoMappingsCache.put(mapKey, autoMapping);
    }
    return autoMapping;
}
```

上面的代码主要进行了如下工作

1. 从`ResultSetWrapper`中获取未配置在`<resultMap>`中的列名
2. 遍历上一步获取到的列名列表
3. 若列名包含列名前缀，则移除前缀，得到属性名
4. 将下划线形式的列名转化为驼峰形式
5. 获取属性类型
6. 获取类型处理器
7. 创建一个`UnMappedColumnAutoMapping`来存储前面获取的信息

这7步中，只有第一步比较难理解，我们继续展开第一步的相关代码，从`4.47`的14行` final List<String> unmappedColumnNames = rsw.getUnmappedColumnNames(resultMap, columnPrefix);`向下。

```java
//4.48
//ResultSetWrapper
public List<String> getUnmappedColumnNames(ResultMap resultMap,
                                           String columnPrefix) throws SQLException {
    
    List<String> unMappedColumnNames 
        = unMappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
    //如果当前还未解析未映射列表(也就是第一次调用此方法)，则进行加载工作
    if (unMappedColumnNames == null) {
        //加载已映射与未映射的列名
        loadMappedAndUnmappedColumnNames(resultMap, columnPrefix);
        //获取未映射的列名
        unMappedColumnNames 
            = unMappedColumnNamesMap.get(getMapKey(resultMap, columnPrefix));
    }
    //如果之前就解析过了直接返回。
    return unMappedColumnNames;
}
```



下面继续从`4.48`的11行`loadMappedAndUnmappedColumnNames(resultMap, columnPrefix);`向下，看看如何加载已映射与未映射的列名

```java
//4.49
//ResultSetWrapper
private void loadMappedAndUnmappedColumnNames(ResultMap resultMap,
                                            String columnPrefix) throws SQLException {
    //创建一个列表来储存已映射列名
    List<String> mappedColumnNames = new ArrayList<>();
    //创建一个列表来储存未映射列名
    List<String> unmappedColumnNames = new ArrayList<>();
    
    
    final String upperColumnPrefix 
        = columnPrefix == null ? null : columnPrefix.toUpperCase(Locale.ENGLISH);
    
	//获取<resultMap>中配置的列名集合
    final Set<String> mappedColumns 
        = prependPrefixes(resultMap.getMappedColumns(), upperColumnPrefix);
    
    //遍历columnNames
    //columnNames是ResultSetWrapper的成员变量，保存了当前结果集中的所有列名
    for (String columnName : columnNames) {
        
        final String upperColumnName = columnName.toUpperCase(Locale.ENGLISH);
        
        //检测已映射Map中是否存在
        if (mappedColumns.contains(upperColumnName)) {
            //存在则存入已映射
            mappedColumnNames.add(upperColumnName);
        
        } else {
            //不存在则存入未映射
            unmappedColumnNames.add(columnName);
        }
    }
    
    //缓存列名集合
    mappedColumnNamesMap
        .put(getMapKey(resultMap, columnPrefix), mappedColumnNames);
    unMappedColumnNamesMap
        .put(getMapKey(resultMap, columnPrefix), unmappedColumnNames);
}
```



下面用一张图举例，看看这个分拣的结果

![](https://i.loli.net/2020/03/27/uofStV1IwcjKkhC.png)



##### 4.2.5.3 映射`<resultMap>`属性到实体类对象

在完成了自动映射后，会将`<resultMap>`中定义过的列名加载到实体类中。我们从代码清单4.42的28行`applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader,columnPrefix)`看看如何进行的映射。



```java
//4.50
//DefaultResultSetHandler
private boolean applyPropertyMappings(ResultSetWrapper rsw,
                                      ResultMap resultMap,
                                      MetaObject metaObject,
                                      ResultLoaderMap lazyLoader,
                                      String columnPrefix)throws SQLException {
    //获取已映射的列名
    final List<String> mappedColumnNames 
        = rsw.getMappedColumnNames(resultMap, columnPrefix);
    
    boolean foundValues = false;
    
    //从resultMap中，获取ResultMapping列表
    final List<ResultMapping> propertyMappings 
        = resultMap.getPropertyResultMappings();
    
    //遍历每个ResultMapping来设置结果集中的数据
    for (ResultMapping propertyMapping : propertyMappings) {
        //拼接列名前缀，得到完整列名
        String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
        
        //如果是在嵌套的结果集，则不处理它
        if (propertyMapping.getNestedResultMapId() != null) {
        // the user added a column attribute to a nested result map, ignore it
            column = null;
        }
        
        //首先检测column是不是复合属性的形式
        //然后检测当前列名是否在已映射列表中，最后是一个多结果集相关的检测
        if (propertyMapping.isCompositeResult()
            || (column != null 
                && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))
            || propertyMapping.getResultSet() != null) {
            
            //从结果集中获取指定列的数据
            Object value 
                = getPropertyMappingValue(rsw.getResultSet(), metaObject,
                                          propertyMapping, lazyLoader, columnPrefix);
            
            // issue #541 make property optional
            
            final String property = propertyMapping.getProperty();
            
            //若列名为空，则不处理它
            if (property == null) {
                continue;
            
            //若获得的值DEFERED，则延迟加载该值    
            } else if (value == DEFERRED) {
                foundValues = true;
                continue;
            }
            
            //若value不为空设置一个标志
            if (value != null) {
                foundValues = true;
            }
            
            //若value不为空，使用元数据类把设置属性到实体类中
            if (value != null 
                || (configuration.isCallSettersOnNulls() 
                    && !metaObject.getSetterType(property).isPrimitive())) {
                                
                metaObject.setValue(property, value);
            }
        }
    }
    return foundValues;
}
```

上述代码主要完成了如下逻辑

1. 首先从`ResultSetWrapper` 中获取已映射列名集合`mappedColumnNames `， 从`ResultMap`获取映射对象`ResultMapping`集合。
2. 然后遍历`ResultMapping`集合
3. 在遍历过程中调用`getPropertyMappingValue`获取指定指定列的数据
4. 然后将获取到的数据设置到实体类对象中。



那么我们接着看看第三步，是怎么获取指定指定列的数据的。

```java
//4.51
//DefaultResultSetHandler
private Object getPropertyMappingValue(ResultSet rs,
                                       MetaObject metaResultObject,
                                       ResultMapping propertyMapping,
                                       ResultLoaderMap lazyLoader,
                                       String columnPrefix)throws SQLException {
    
    //与关联查询有关，此处不解释
    if (propertyMapping.getNestedQueryId() != null) {
        return getNestedQueryMappingValue(rs, metaResultObject,
                                          propertyMapping, lazyLoader, columnPrefix);
    
    //延迟加载相关，不解释
    } else if (propertyMapping.getResultSet() != null) {
        addPendingChildRelation(rs, metaResultObject, propertyMapping);   
        return DEFERRED;
    
    //正常情况    
    } else {
        //从ResultMapping中拿到typeHandler
        final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
        
        //获取对应的列名
        final String column 
            = prependPrefix(propertyMapping.getColumn(), columnPrefix);
        
        //调用typeHandler来获取指定的值
        return typeHandler.getResult(rs, column);
    }
}
```

通过4.2.5.1-4.2.5.3，我们完成了实体类的创建，并向其中设置了各种属性。下面两个小节，我们看一看MyBatis如何实现关联查询和延迟加载的。



##### 4.2.5.4 关联查询



我们在学习 MyBatis框架时，会经常碰到一对一，一对多的使用场景。对于这样的场景，我们可以使用关联查询，将一条SQL拆成两条去完成查询任务。 MyBatis提供了两个标签用于支持一对一和一对多的使用场景，分别是 `<association>`和`<collection>`。



下面来看一看源码，上接代码清单`4.51`的第9行`return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);`

```java
//4.52
//DefaultResultSetHandler
private Object getNestedQueryMappingValue(ResultSet rs, 
                                          MetaObject metaResultObject,
                                          ResultMapping propertyMapping,
                                          ResultLoaderMap lazyLoader,
                                          String columnPrefix) throws SQLException {
    //获得关联查询的id，id = 命名空间 + <association>中的select属性值
    final String nestedQueryId = propertyMapping.getNestedQueryId();
    //获取关联查询上的属性名，<association>中的property属性值
    final String property = propertyMapping.getProperty();
    //根据nestedQueryId获取这个关联查询对应的MappedStatement
    final MappedStatement nestedQuery 
        = configuration.getMappedStatement(nestedQueryId);
    
    final Class<?> nestedQueryParameterType = nestedQuery.getParameterMap().getType();
    //生成关联查询的参数对象，例如
    //1. <association column="article_id">，此时就是个Integer对象
    //2. <association column="{id=article_id, name=title}"，此时就是一个实体类或者Map
    final Object nestedQueryParameterObject 
        = prepareParameterForNestedQuery(rs,
                                         propertyMapping,
                                         nestedQueryParameterType,
                                         columnPrefix);
    
    Object value = null;
    
    if (nestedQueryParameterObject != null) {
        
        //获取BoundSql
        final BoundSql nestedBoundSql 
            = nestedQuery.getBoundSql(nestedQueryParameterObject);
        
        //创建CacheKey
        final CacheKey key 
            = executor.createCacheKey(nestedQuery, nestedQueryParameterObject,
                                      RowBounds.DEFAULT, nestedBoundSql);
        
        final Class<?> targetType = propertyMapping.getJavaType();
        
        //检查一级缓存中是否保存了关联查询的结果，有的话通过MataObject将结果放入到对应实体类中
        if (executor.isCached(nestedQuery, key)) {
            executor.deferLoad(nestedQuery, metaResultObject,
                               property, key, targetType);
            value = DEFERRED;
        //如果一级缓存中没有
        } else {
            //创建结果加载器
            final ResultLoader resultLoader 
                = new ResultLoader(configuration, executor, nestedQuery,
                                   nestedQueryParameterObject, targetType,
                                   key, nestedBoundSql);
            //如果当前属性需要延迟加载
            if (propertyMapping.isLazy()) {
                //添加延迟加载的相关对象到loaderMap集合中
                lazyLoader.addLoader(property, metaResultObject, resultLoader);
                value = DEFERRED;
            //如果不需要延迟加载
            } else {
                //直接加载结果
                value = resultLoader.loadResult();
            }
        }
    }
    //返回标志，代表是否加载了
    return value;
}
```

下面总结这个方法的逻辑

1. 根据`nestedQueryId`获取`MappedStatement`
2. 生成参数对象
3. 获取`BoundSql`
4. 检查一级缓存中是否有关联查询的结果。有，则将结果设置到实体类对象中
5. 若一级缓存中没有，则创建结果加载器`ResultLoader`
6. 检查当前属性是否需要进行延迟加载。若需要，则添加延迟加载相关的对象到`loaderMap`集合中。等待真正需要的时候再进行加载。
7. 如不需要延迟加载，则直接通过结果加载器`ResultLoader`加载结果。

##### 4.2.5.5 延迟加载

这一小节紧接上一小节，看看延迟加载的一些源码，上接代码清单`4.52`的55行`lazyLoader.addLoader(property, metaResultObject, resultLoader);`，我们看看添加延迟加载相关对象到`loaderMap`集合中的逻辑。

````java
//4.53
//ResultLoaderMap

//一个Map，用来存储延迟加载的对象
private final Map<String, LoadPair> loaderMap = new HashMap<>();

public void addLoader(String property,
                      MetaObject metaResultObject,
                      ResultLoader resultLoader) {
    
    //将属性名转化为大写
    String upperFirst = getUppercaseFirstProperty(property);
    
    //如果已经存在就抛出异常
    if (!upperFirst.equalsIgnoreCase(property) 
        && loaderMap.containsKey(upperFirst)) {
      throw new ExecutorException("Nested lazy loaded result property '" 
                                  + property
                                  + "' for query id '" 
                                  + resultLoader.mappedStatement.getId()
              					  + " already exists in the result map. ");
    }
    
    //创建一个LoaderPair，并将大写的属性名作为键，LoaderPair作为值存放到LoaderMap中
    loaderMap.put(upperFirst, new LoadPair(property, metaResultObject, resultLoader));
}
````



`addLoader`方法的参数最终都传给了`LoadPair`。该类的`load`方法会在内部调用`ResultLoader`的`loadResult`方法进行关联查询，并通过`metaResultObject`将查询结果设置到实体类对象中。那么`LoadPair`的`load`方法由谁来调用呢？答案是实体类的代理对象。



再分析源码之前，我们先来看看如何开启`lazyLoad`吧

```xml
<!--开启延迟加载-->
<setting name="lazyLoadingEnabled" value="true"/>
<!--关闭积极的延迟加载策略-->
<setting name="aggressiveLazyLoading" value="false"/>
<!--延迟加载的触发方法-->
<setting name="lazyloadTriggerMethods" value="equal,hashCode"/>
```



MyBatis会为需要延迟加载的类生成代理类，代理逻辑会拦截实体类的方法调用。默认情况下，MyBatis会使用`Javassist`为实体类生成代理，代理逻辑封装在`JavaassitProxyFactory`类中。

```java
//4.54
//JavaassitProxyFactory
public Object invoke(Object enhanced, Method method, Method methodProxy, Object[] args) throws Throwable {
    final String methodName = method.getName();
    try {
        synchronized (lazyLoader) {
            //针对writeReplace方法的处理逻辑，与延迟加载无关，跳过
            if (WRITE_REPLACE_METHOD.equals(methodName)) {
                Object original;
                if (constructorArgTypes.isEmpty()) {
                        original = objectFactory.create(type);
                } else {
                        original = objectFactory
                            .create(type, constructorArgTypes, constructorArgs);
                }
                PropertyCopier.copyBeanProperties(type, enhanced, original);
                if (lazyLoader.size() > 0) {
                        return new JavassistSerialStateHolder(original,	   
                                                           lazyLoader.getProperties(), 
                                                              objectFactory, 
                                                              constructorArgTypes, 
                                                              constructorArgs);
                } else {
                        return original;
                }
             //针对延迟加载的处理逻辑   
            } else {
                
                if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
                    
                    //如果aggressive为true或者触发方法(equals、hashcode等)被调用，
                    if (aggressive 
                            || lazyLoadTriggerMethods.contains(methodName)) {
                        //则加载延迟加载的数据   
                        lazyLoader.loadAll();
                    
                    //如果使用者调用的是setter方法，将相应的延迟加载类从loadMap中移除    
                    } else if (PropertyNamer.isSetter(methodName)) {
                            
                        final String property 
                                = PropertyNamer.methodToProperty(methodName);  
                        lazyLoader.remove(property);
                    
                    //如果使用者调用的是getter方法，将相应的延迟加载类从loadMap中移除     
                    } else if (PropertyNamer.isGetter(methodName)) {
                        final String property 
                                = PropertyNamer.methodToProperty(methodName);
                        //检测该属性是否有相应的`LoadPair`对象，若有
                        //则执行延迟加载逻辑
                        if (lazyLoader.hasLoader(property)) {
                            lazyLoader.load(property);   
                        }    
                    }
                }
            }
        }
        //调用被代理类本想执行的方法
        return methodProxy.invoke(enhanced, args);
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
}
```

我们还可以从46行向下，看看实际的加载逻辑，但是这里就不继续了。



##### 4.2.5.6 存储映射结果



这部分的代码比较简单，它是从代码清单`4.40`的第23行`storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);`向下的。这里不分析，仅仅贴出源码。



```java
//4.55
//DefaultResultSetHandler
private void storeObject(ResultHandler<?> resultHandler,
                         DefaultResultContext<Object> resultContext,
                         Object rowValue,
                         ResultMapping parentMapping,
                         ResultSet rs) throws SQLException {
    //处理多结果集的情况
    if (parentMapping != null) {
        
        linkToParents(rs, parentMapping, rowValue);
    //处理单结果集的情况
    } else {
        //直接调用
        callResultHandler(resultHandler, resultContext, rowValue);
    
    }
}

//上接15行的调用  
private void callResultHandler(ResultHandler<?> resultHandler,
                               DefaultResultContext<Object> resultContext,
                               Object rowValue) {
    //设置结果到resultContext中
    resultContext.nextResultObject(rowValue);
    //从resultContext获取结果，并存储到resultHandler中
    ((ResultHandler<Object>) resultHandler).handleResult(resultContext);
}
```



```java
//4.56
//DefaultResultContext
//上接代码清单4.55的25行
public class DefaultResultContext<T> implements ResultContext<T> {

  private T resultObject;
  private int resultCount;
  private boolean stopped;

  public DefaultResultContext() {
    resultObject = null;
    resultCount = 0;
    stopped = false;
  }

  @Override
  public T getResultObject() {
    return resultObject;
  }

  @Override
  public int getResultCount() {
    return resultCount;
  }

  @Override
  public boolean isStopped() {
    return stopped;
  }

  public void nextResultObject(T resultObject) {
    resultCount++;
    this.resultObject = resultObject;
  }

  @Override
  public void stop() {
    this.stopped = true;
  }

}
```



```java
//4.57
//DefaultResultHandler
//上接代码清单4.55的27行
public class DefaultResultHandler implements ResultHandler<Object> {

    private final List<Object> list;
  
    public DefaultResultHandler() {
        list = new ArrayList<>();
    }
  
    @SuppressWarnings("unchecked")
    public DefaultResultHandler(ObjectFactory objectFactory) {
        list = objectFactory.create(List.class);
    }
  
    @Override
    public void handleResult(ResultContext<?> context) {
        list.add(context.getResultObject());
    }
  
    public List<Object> getResultList() {
        return list;
    }
  
}
```



### 4.3 SQL执行过程总结



经过前面前面的分析，相信大家对MyBatis 执行SQL 的过程都有比较深入的理解。本章的最后，用一张图MyBatis 的执行过程进行一个总结。如下：



![](https://i.loli.net/2020/03/27/vF7xszwUi5d4olX.png)




在MyBatis 中，SQL 执行过程的实现代码是有层次的，每层都有相应的功能。

1. `SqlSession` 是对外接口的接口，因此它提供了各种语义清晰的方法，供使用者调用。
2. `Executor`层做的事情较多，比如一二级缓存功能就是嵌入在该层内的。
3. `StatementHandler` 层主要是与`JDBC` 层面的接口打交道。
4. 至于`ParameterHandler` 和`ResultSetHandler`，一个负责向SQL 中设置运行时参数，另一个负责处理SQL 执行结果，它们俩可以看做是`StatementHandler `辅助类。
5. 最后看一下右边横跨数层的类，`Configuration` 是一个全局配置类，很多地方都依赖它。`MappedStatement` 对应SQL 配置，包含了SQL 配置的相关信息。`BoundSql `中包含了已完成解析的SQL 语句，以及运行时参数等。到此，关于SQL 的执行过程就分析完了。