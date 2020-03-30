---
title: '[MyBatis源码][7][插件机制]'
date: 2020-03-30 21:16:34
tags:
    - MyBatis
categories:
    - MyBatis
---
## 第七章 插件机制



一般情况下，开源框架都会提供插件或其他形式的拓展点，供开发者自行拓展。以 MyBatis 为例，我们可基于 MyBatis 插件机制实现分页、分表，监控等功能。由于插件和业务无关，业务也无法感知插件的存在。因此可以无感植入插件，在无形中增强功能。

### 7.1 插件机制原理

我们在编写插件时，除了需要让插件类实现`Interceptor`接口外，还需要通过注解标注该插件的拦截点。所谓拦截点指的是插件所能拦截的方法，MyBatis 所允许拦截的拦截点如下：

| 类名               | 方法名                                                       |
| ------------------ | ------------------------------------------------------------ |
| `Executor`         | `update`, `query`,` flushStatements`,`commit`, `rollback`,`getTransaction`, `close`, `isClosed` |
| `ParameterHandler` | `getParameterObject`, `setParameters`                        |
| `ResultSetHandler` | `handleResultSets`, `handleOutputParameters`                 |
| `StatementHandler` | `prepare`, `parameterize`, `batch`, `update`, `query`        |



下面举个例子，假如我们想要拦截`Executor`的`query`方法，可以这样定义插件

```java
//id:7.1
@Intercepts({
    @Signature(
        type = Executor.class,
        method = "query",
        args ={MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
	)
})
public class ExamplePlugin implements Interceptor {
// 逻辑省略
}
```



### 7.2 植入插件逻辑



下面我们以`Executor`为例，分析`MyBatis`是如何为`Executor`实例植入插件逻辑的。其实，原理还是动态代理。先看一看`SqlSession`的开启逻辑。



```java
//id:7.2
//DefaultSqlSession
public SqlSession openSession() {
    
    return openSessionFromDataSource(
        configuration.getDefaultExecutorType(), null, false);

}


private SqlSession openSessionFromDataSource(ExecutorType execType,
                                             TransactionIsolationLevel level,
                                             boolean autoCommit) {
    
    Transaction tx = null;
    
    try {
        
        //与本章内容无关
        final Environment environment = configuration.getEnvironment();
        final TransactionFactory transactionFactory 
            = getTransactionFactoryFromEnvironment(environment);
        tx = transactionFactory.newTransaction(
            environment.getDataSource(), level, autoCommit);
        
        //创建Executor
        final Executor executor = configuration.newExecutor(tx, execType);
        
        return new DefaultSqlSession(configuration, executor, autoCommit);
    
    } catch (Exception e) {
        
        closeTransaction(tx); // may have fetched a connection so lets call close()
        
        throw ExceptionFactory
            .wrapException("Error opening session.  Cause: " + e, e);
    
    } finally {
    
        ErrorContext.instance().reset();
    
    }

}  
```



接着从第26行`final Executor executor = configuration.newExecutor(tx, execType);`向下，看看如何创建`Executor`

```java
//id:7.3
//Configuration
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    
    //设置一个executorType，若为null，则为defaultExecutorType
    executorType = executorType == null ? defaultExecutorType : executorType;
    
    //若defaultExecutorType也没设置，则为SIMPLE
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    
    //根据不同的类型创建不同的Executor
    if (ExecutorType.BATCH == executorType) {
        executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
        executor = new ReuseExecutor(this, transaction);
    } else {
        executor = new SimpleExecutor(this, transaction);
    }
    
    //根据是否开启了二级缓存，来决定是否用CachingExecutor包裹
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);
    }
    
    //植入插件
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```



接着从26行 `executor = (Executor) interceptorChain.pluginAll(executor);`向下，看看是如何植入插件链中的所有插件的。



```java
//id:7.4
//InterceptorChain
public class InterceptorChain {

    private final List<Interceptor> interceptors = new ArrayList<>();
  
    //植入所有插件
    public Object pluginAll(Object target) {
      for (Interceptor interceptor : interceptors) {
        //其实就是调用每个interceptor的plugin方法  
        target = interceptor.plugin(target);
      }
      return target;
    }
  
    //向列表中添加插件实例
    public void addInterceptor(Interceptor interceptor) {
      interceptors.add(interceptor);
    }
  
    //获取插件列表
    public List<Interceptor> getInterceptors() {
      return Collections.unmodifiableList(interceptors);
    }
  
}
```



`interceptor`的`plugin`方法由具体的插件类实现，不过该方法代码一般比较固定，下面找一个实例分析下

```java
//id:7.5
//ExampleInterpretor
public Object plugin(Object target){
    return Plugin.wrap(target, this);
}
```

这是最简单的一种实现方式，它直接调用辅助类`Plugin`的`wrap`方法。我们来看看这个方法。

```java
//id:7.6
//Plugin
public static Object wrap(Object target, Interceptor interceptor) {
    
    //解析插件类@Signature注解内容，并根据Signature中的各个属性，生成一条条映射。
    //键为拦截点的类名，值为拦截点要拦截的各个方法组成的列表
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    
    //获取被代理者的类型
    Class<?> type = target.getClass();
    
    //获取被代理者实现的接口列表
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    
    //如果它实现了一些接口，则通过Proxy.newProxyInstance创建
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    
    //如果它没有实现任何接口，直接返回。
    return target;
}
```

如上，`plugin` 方法在内部调用了`Plugin` 类的`wrap` 方法，用于为目标对象生成代理。`Plugin`类实现了`InvocationHandler` 接口，因此它可以作为参数传给`Proxy` 的`newProxyInstance` 方法。



此外，假如有多个拦截器，则会多次调用`plugin`方法，最终生成一个层层嵌套的代理类。如下图所示

![](https://i.loli.net/2020/03/30/1OR3xXTZNsceAHa.png)

当`Executor`的某个方法执行时，插件逻辑会先行执行。执行顺序由外而内，比如上图的执行顺序为`plugin3->plugin2->plugin1->Executor`。



### 7.3 执行插件逻辑

`Plugin` 实现了 `InvocationHandler` 接口，因此它的 `invoke` 方法会拦截所有的方法调用。`invoke` 方法会对所拦截的方法进行检测，以决定是否执行插件逻辑。该方法的逻辑如下：

```java
//id:7.7
//Plugin
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        //获取这个interpretor拦截的方法列表
        Set<Method> methods = signatureMap.get(method.getDeclaringClass());
        
        //检测被拦截的方法方法是否在列表里，若在
        if (methods != null && methods.contains(method)) {
            //则执行插件逻辑
            return interceptor.intercept(new Invocation(target, method, args));
        }
        
        //执行被拦截的方法本身的逻辑
        return method.invoke(target, args);
    } catch (Exception e) {
        throw ExceptionUtil.unwrapThrowable(e);
    }
}
```



`invoke` 方法的代码比较少，逻辑不难理解。

1. 首先，`invoke` 方法会检测被拦截方法是否配置在插件的 `@Signature` 注解中
2. 若是，则执行插件逻辑
3. 最后，执行被拦截方法。



插件逻辑封装在`intercept` 中，该方法的参数类型为`Invocation`。`Invocation` 主要用于存储目标类，方法以及方法参数列表。下面简单看一下该类的定义。

```java
//id:7.8
//Invocation
public class Invocation {

    private final Object target;
    private final Method method;
    private final Object[] args;
  
    public Invocation(Object target, Method method, Object[] args) {
      this.target = target;
      this.method = method;
      this.args = args;
    }
  
    public Object getTarget() {
      return target;
    }
  
    public Method getMethod() {
      return method;
    }
  
    public Object[] getArgs() {
      return args;
    }
  
    public Object proceed() throws InvocationTargetException, IllegalAccessException {
      return method.invoke(target, args);
    }
  
}
```



到此，MyBatis插件机制就分析完了。