---
title: '[MyBatis源码][5][连接池]'
date: 2020-03-28 17:10:54
tags:
    - MyBatis
categories:
    - MyBatis
---
## 第五章 连接池



数据源主要用来获取数据库连接，本篇文章将向大家介绍 MyBatis 内置数据源的实现逻辑。MyBatis 支持三种数据源配置，分别为`UNPOOLED`、`POOLED`和`JNDI`。并提供了两种数据源实现，分别是`UnpooledDataSource`和`PooledDataSource`。



### 5.1 内置数据源配置和初始化过程

在分析数据源之前，先看一下数据源如何配置

```xml
<environment>
    <dataSource type="POOLED|UNPOOLED">
        <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
        <property name="url" value="jdbcc:mysql//..."/>
        <property name="username" value="root"/>
        <property name="password" value="1234"/>
    </dataSource>
</environment>
```

数据源的配置是内嵌在`<environment>`节点中的，MyBatis 在解析`<environment>`节点时，
会一并解析数据源的配置。MyBatis 会根据具体的配置信息，为不同的数据源创建相应工厂
类，通过工厂类即可创建数据源实例。



接下来我们就从对于`<environment>`节点的解析方法`XMLConfigBuilder.environmentsElement`开始源码分析之路

```java
//id:5.1
//XMLConfigBuilder
private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
        if (environment == null) {
            environment = context.getStringAttribute("default");
        }
        for (XNode child : context.getChildren()) {
            String id = child.getStringAttribute("id");
            
            if (isSpecifiedEnvironment(id)) {
                //解析事务管理器
                TransactionFactory txFactory 
                    = transactionManagerElement(child.evalNode("transactionManager"));
                //解析DataSource节点生成DataSourceFactory
                DataSourceFactory dsFactory 
                    = dataSourceElement(child.evalNode("dataSource"));
                //从工厂中生产DataSource
                DataSource dataSource = dsFactory.getDataSource();
                //使用Builder构建一个Environment实例，实例中包含一个DataSource属性
                Environment.Builder environmentBuilder = new Environment.Builder(id)
                    .transactionFactory(txFactory)
                    .dataSource(dataSource);
                //将建好的Environment存放到Configuration中
                configuration.setEnvironment(environmentBuilder.build());
            }
        }
    }
}
```

可以看到如何创建`DataSourceFactory`的逻辑在第14行的`dataSourceElement(child.evalNode("dataSource"));`中，我们继续打开它



```java
//id:5.2
//XMLConfigBuilder
private DataSourceFactory dataSourceElement(XNode context) throws Exception {
    if (context != null) {
        //获取<dataSource>标签的type属性
        String type = context.getStringAttribute("type");
        //将<dataSource>标签的所有子标签解析为一个Properties
        Properties props = context.getChildrenAsProperties();
        //通过type来反射创建指定类型的dataSource
        DataSourceFactory factory 
            = (DataSourceFactory) resolveClass(type).newInstance();
        //将这些属性设置到工厂中
        factory.setProperties(props);
        //返回
        return factory;
    }
    throw new BuilderException("Environment declaration requires a DataSourceFactory.");
}
```



下面我们来看看DataSourceFactory的真面目，它是一个接口，它有两个实现`PooledDataSourceFactory`和`UnpooledDataSourceFactory`。

```java
//5.3
//DataSourceFactory
public interface DataSourceFactory {
	//为数据源设置属性
    void setProperties(Properties props);
  	//返回数据源
    DataSource getDataSource();
  
}
```



然后看一看它的第一个实现类`PooledDataSourceFactory`

```java
//5.4
//UnpooledDataSourceFactory
public class UnpooledDataSourceFactory implements DataSourceFactory {

    private static final String DRIVER_PROPERTY_PREFIX = "driver.";
    
    private static final int DRIVER_PROPERTY_PREFIX_LENGTH 
        = DRIVER_PROPERTY_PREFIX.length();
  
    protected DataSource dataSource;
  
    public UnpooledDataSourceFactory() {
        //这个工厂初始化的是一个UnpooledDataSource对象
        this.dataSource = new UnpooledDataSource();
    }
  
    //为这个DataSource设置属性
    @Override
    public void setProperties(Properties properties) {
        
        Properties driverProperties = new Properties();
        //获取DataSource类的元信息实例
        MetaObject metaDataSource = SystemMetaObject.forObject(dataSource);
        
        //遍历传入的Property
        for (Object key : properties.keySet()) {
            
            //获取Property的名字
            String propertyName = (String) key;
            
            //如果名字是"driver."开头的，则
            if (propertyName.startsWith(DRIVER_PROPERTY_PREFIX)) {
                
                String value = properties.getProperty(propertyName);
                //将这个property的映射加入到driverProperties中
                driverProperties.setProperty(
                    propertyName.substring(DRIVER_PROPERTY_PREFIX_LENGTH),
                    value);
            
            //如果名字不是"driver."开头的，并且DataSource中有这个属性    
            } else if (metaDataSource.hasSetter(propertyName)) {
                //取得属性值
                String value = (String) properties.get(propertyName);
                //根据DataSource中设置的这个属性的类型进行类型转换
                Object convertedValue 
                    = convertValue(metaDataSource, propertyName, value);
                //调用元信息类来设置属性
                metaDataSource.setValue(propertyName, convertedValue);
            //处理异常情况
            } else {
                throw new DataSourceException("Unknown DataSource property: " 
                                              + propertyName);
            }
        }
        //将driverProperties设置到DataSource中
        if (driverProperties.size() > 0) {
            metaDataSource.setValue("driverProperties", driverProperties);
        }
    }
  
    //直接返回这个DataSource实例即可
    @Override
    public DataSource getDataSource() {
      return dataSource;
    }
  
    //对property进行类型转换的方法，略
    private Object convertValue(MetaObject metaDataSource,
                                String propertyName, String value) {
        
        Object convertedValue = value;
        
        Class<?> targetType = metaDataSource.getSetterType(propertyName);
        
        if (targetType == Integer.class || targetType == int.class) {
            convertedValue = Integer.valueOf(value);
        
        } else if (targetType == Long.class || targetType == long.class) {
            convertedValue = Long.valueOf(value);
        
        } else if (targetType == Boolean.class || targetType == boolean.class) {
            convertedValue = Boolean.valueOf(value);
        }
        return convertedValue;
    }
  
}
```



然后我们再来看一看第二个实现类`PooledDataSourceFactory`，这个类继承自`UnpooledDataSourceFactory`只是在构造器初始化`DataSource`时，使用的是`PooledDataSource`的实例而已。仅此而已，其他方法的实现完全继承自父类。

```java
//5.5
//PooledDataSourceFactory
public class PooledDataSourceFactory extends UnpooledDataSourceFactory {

  public PooledDataSourceFactory() {
      this.dataSource = new PooledDataSource();
  }

}
```



我们就是通过这两个工厂来创建两种类型的`DataSource`，完成创建后返回的`DataSource`实例，将会被存储在`Environment`里，而`Environment`又会被存储入`Configuration`。于是，当我们要使用`DataSource`时，只需要调用`configuration.getDataSource()`即可。



### 5.2 UnpooledDataSource



`UnpooledDataSource`，从名称上即可知道，该种数据源不具有池化特性。该种数据源每次会返回一个新的数据库连接，而非复用旧的连接。



下面来看看如何通过它来获取数据库连接

```java
//5.6
//PooledDataSourceFactory
private Connection doGetConnection(Properties properties) throws SQLException {
    //初始化数据库驱动
    initializeDriver();
    //通过数据库驱动获得数据库连接
    Connection connection = DriverManager.getConnection(url, properties);
    //配置数据库连接
    configureConnection(connection);
    return connection;
}
```



如上，获取数据库连接总共分为3步

1. 初始化数据库驱动，详见5.2.1
2. 通过数据库驱动获得数据库连接
3. 配置数据库连接，详见5.2.3

#### 5.2.1 初始化数据库驱动

下面我们展开代码清单`5.6`的第五行` initializeDriver();`，看看如何初始化数据库驱动。

```java
//5.7
//PooledDataSourceFactory
private synchronized void initializeDriver() throws SQLException {
    //如果缓存中找不到这个driver的实例，也就是没有注册过，就执行注册，否则就不用再注册了
    if (!registeredDrivers.containsKey(driver)) {
        
        Class<?> driverType;
        
        try {
            //加载驱动类型
            if (driverClassLoader != null) {
                driverType = Class.forName(driver, true, driverClassLoader);
            } else {
                driverType = Resources.classForName(driver);
            }
            //通过反射创建驱动实例
            Driver driverInstance = (Driver)driverType.newInstance();
            //注册这个驱动
            DriverManager.registerDriver(new DriverProxy(driverInstance));
            //缓存驱动类名和实例
            registeredDrivers.put(driver, driverInstance);
        
        } catch (Exception e) {
            throw new SQLException("Error on UnpooledDataSource. Cause: " + e);
        }
    }
}
```

如上，`initializeDriver`方法主要包含三步操作，分别如下：
1. 加载驱动
2. 通过反射创建驱动实例
3. 注册驱动实例
4. 缓存驱动，上面代码中出现了缓存相关的逻辑，这个是用于避免重复注册驱动



#### 5.2.2 配置数据库连接



接下来我们再看看从driver中获取连接之后，需要对连接进行哪些配置

```java
//5.8
//PooledDataSourceFactory
private void configureConnection(Connection conn) throws SQLException {
    //设置自动提交
    if (autoCommit != null && autoCommit != conn.getAutoCommit()) {
        conn.setAutoCommit(autoCommit);
    }
    //设置数据库事务的隔离级别
    if (defaultTransactionIsolationLevel != null) {
        conn.setTransactionIsolation(defaultTransactionIsolationLevel);
    }
}
```



### 5.3 PooledDataSource

`PooledDataSource`内部实现了连接池功能，用于复用数据库连接。因此，从效率上来说，`PooledDataSource` 要高于`UnpooledDataSource`。



#### 5.3.1 辅助类介绍



`PooledDataSource`需要借助两个辅助类帮其完成功能，这两个辅助类分别是`PoolState`和`PooledConnection`。

- `PoolState`用于记录连接池运行时的状态，比如连接获取次数，无效连接数量等。同时`PoolState`内部定义了两个`PooledConnection`集合，用于存储空闲连接和活跃连接。

- `PooledConnection`内部定义了一个`Connection`类型的变量，用于指向真实的数据库连接。以及一个 `Connection`的代理类，用于对部分方法调用进行拦截。至于为什么要拦截，随后将进行分析。除此之外，`PooledConnection`内部也定义了一些字段，用于记录数据库连接的一些运行时状态。

  

接下来，我们来看一下 `PooledConnection` 的定义。


```java
//id:5.9
//PooledConnection
class PooledConnection implements InvocationHandler {

    private static final String CLOSE = "close";
    
    private static final Class<?>[] IFACES = new Class<?>[] { Connection.class };
  
    private final int hashCode;
    
    //指向池化数据源的一个引用
    private final PooledDataSource dataSource;
    
    //真实的数据库连接
    private final Connection realConnection;
    
    //数据库连接代理
    private final Connection proxyConnection;
    
    //从连接池中取出连接时的时间戳
    private long checkoutTimestamp;
    
    //连接创建的时间戳
    private long createdTimestamp;
    
    //连接上次使用的时间戳
    private long lastUsedTimestamp;
    
    //(url + username + password).hashcode()
    private int connectionTypeCode;
    
    //表示连接是否有效
    private boolean valid;
  	
    public PooledConnection(Connection connection, PooledDataSource dataSource) {
        this.hashCode = connection.hashCode();
        this.realConnection = connection;
        this.dataSource = dataSource;
        this.createdTimestamp = System.currentTimeMillis();
        this.lastUsedTimestamp = System.currentTimeMillis();
        this.valid = true;
        //创建Connection的代理类对象
        this.proxyConnection
            = (Connection) Proxy.newProxyInstance(Connection.class.getClassLoader(),
                                                  IFACES, this);
    }
    
    //动态代理的实际执行方法，在执行真正的连接对象的方法之前和之后，插入一些逻辑
    @Override
    public Object invoke(Object proxy, Method method,
                         Object[] args) throws Throwable {
        
        String methodName = method.getName();
        
        if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
            dataSource.pushConnection(this);
            return null;
        }
        
        try {
            if (!Object.class.equals(method.getDeclaringClass())) {
                // issue #579 toString() should never fail
                // throw an SQLException instead of a Runtime
                checkConnection();
            }
            return method.invoke(realConnection, args);
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
  
    }
   
}
```



下面我们再来看看`PoolState`的定义

```java
//id:5.10
//PoolState
public class PoolState {
	
    //指向池化数据源的一个引用
    protected PooledDataSource dataSource;
  
    //空闲连接列表
    protected final List<PooledConnection> idleConnections = new ArrayList<>();
    
    //活跃连接列表
    protected final List<PooledConnection> activeConnections = new ArrayList<>();
    
    //从连接池中获取连接的次数
    protected long requestCount = 0;
    
    //请求连接总耗时
    protected long accumulatedRequestTime = 0;
    
    //请求执行时间总耗时
    protected long accumulatedCheckoutTime = 0;
    
    //执行时间超时的连接数
    protected long claimedOverdueConnectionCount = 0;
    
    //超时时间累加值
    protected long accumulatedCheckoutTimeOfOverdueConnections = 0;
    
    //等待时间累加值
    protected long accumulatedWaitTime = 0;
    
    //等待次数
    protected long hadToWaitCount = 0;
    
    //无效连接数
    protected long badConnectionCount = 0;
} 
```



#### 5.3.2 获取连接

下面看一下`PooledDataSource`如何获取连接，此处的逻辑比较复杂，先给出流程图

![](https://i.loli.net/2020/03/28/2FjsBS6N1oxbqAX.png)

然后我们用一段伪代码演示这个流程图

```
//id:5.11
if(连接池中有空闲连接){
	1. 将连接从空闲连接集合中移除
}else{
	if(活跃连接数未超出限制){
		1. 创建新连接
	}else{
		1. 从活跃连接集合中取出第一个元素
		2. 获取连接运行时长
		if(连接超时){
			1. 将连接从活跃集合中删除
			2. 复用原连接的成员变量，并创建新的PooledConnection对象
		}else{
			1. 线程进入等待状态
			2. 线程被唤醒，重新执行以上逻辑
		}
	}
}
1. 将连接添加到活跃连接集合中
2. 返回连接
```



最后，我们再来看源码

```java
//id:5.12
//PooledDataSource
public Connection getConnection() throws SQLException {
    //返回Connection的代理对象
    return popConnection(dataSource.getUsername(),
                         dataSource.getPassword()).getProxyConnection();
}


private PooledConnection popConnection(String username, String password) throws SQLException {
    boolean countedWait = false;
    PooledConnection conn = null;
    long t = System.currentTimeMillis();
    int localBadConnectionCount = 0;

    while (conn == null) {
        //加锁
        synchronized (state) {
            //如果有空闲的连接
            if (!state.idleConnections.isEmpty()) {
                
                //从空闲连接列表中移除第一个
                // Pool has available connection
                conn = state.idleConnections.remove(0);
                
                //打印信息到logger
                if (log.isDebugEnabled()) {
                    log.debug("Checked out connection " 
                              + conn.getRealHashCode() + " from pool.");
                }
                
            //如果没有空闲连接    
            } else {
                
                //如果活跃的连接数小于允许的最大活跃连接数，则
                // Pool does not have available connection
                if (state.activeConnections.size() < poolMaximumActiveConnections) {
                    
                    //创建一个新的连接
                    // Can create new connection
                    conn = new PooledConnection(dataSource.getConnection(), this);
                    
                    if (log.isDebugEnabled()) {
                        log.debug("Created connection " 
                                  + conn.getRealHashCode() + ".");
                    }
                //如果活跃的连接数大于允许的最大活跃连接数（此时不能创建新的连接）    
                } else {
                    // Cannot create new connection
                    
                    //从活跃连接列表中取出第一个连接
                    PooledConnection oldestActiveConnection 
                        = state.activeConnections.get(0);
                    
                    
                    long longestCheckoutTime =
                        oldestActiveConnection.getCheckoutTime();
                    
                    //检查它是否超时，如果已经超时了
                    if (longestCheckoutTime > poolMaximumCheckoutTime) {
                        // Can claim overdue connection
                        
                        //更新一些state中的数据
                        state.claimedOverdueConnectionCount++;
                        state.accumulatedCheckoutTimeOfOverdueConnections 
                            += longestCheckoutTime;
                        state.accumulatedCheckoutTime += longestCheckoutTime;
                        
                        //从活跃连接列表中移除这个连接
                        state.activeConnections.remove(oldestActiveConnection);
                        
                        //回滚这个连接正在进行的操作
                        if (!oldestActiveConnection
                            .getRealConnection().getAutoCommit()) {
                            
                            try {
                                oldestActiveConnection.
                                    getRealConnection().rollback();
                            } catch (SQLException e) {
                                log.debug("Bad connection. Could not roll back");
                            }
                        }
                        
                        //用这个老的连接里面的realConnection新建一个PooledConnection
                        conn = new PooledConnection(oldestActiveConnection.
                                                    getRealConnection(), this);
                        
                        //设置一些属性
                        conn.setCreatedTimestamp(
                            oldestActiveConnection.getCreatedTimestamp());
                        conn.setLastUsedTimestamp(
                            oldestActiveConnection.getLastUsedTimestamp());
                        
                        //设置老的连接为无效状态
                        oldestActiveConnection.invalidate();
                        
                        if (log.isDebugEnabled()) {
                            log.debug("Claimed overdue connection " 
                                      + conn.getRealHashCode() + ".");
                        }
                    
                    //如果没有超时    
                    } else {
                    // Must wait
                        try {
                            if (!countedWait) {
                                state.hadToWaitCount++;
                                countedWait = true;
                            }
                            if (log.isDebugEnabled()) {
                                log.debug("Waiting as long as " 
                                          + poolTimeToWait 
                                          + " milliseconds for connection.");
                            }
                            
                            long wt = System.currentTimeMillis();
                            
                            //挂起当前线程
                            state.wait(poolTimeToWait);
                            
                            state.accumulatedWaitTime 
                                += System.currentTimeMillis() - wt;
                        
                        } catch (InterruptedException e) {
                            break;
                        }
                    }
                }
            }
            
            //取到连接后
            if (conn != null) {
                //检查它是否有效，如果有效
                // ping to server and check the connection is valid or not
                if (conn.isValid()) {
                    //先回滚连接中未提交的操作
                    if (!conn.getRealConnection().getAutoCommit()) {
                        conn.getRealConnection().rollback();
                    }
                    
                    //设置一些属性
                    conn.setConnectionTypeCode(
                        assembleConnectionTypeCode(
                            dataSource.getUrl(), username, password));
                 	
                    conn.setCheckoutTimestamp(System.currentTimeMillis());
                    
                    conn.setLastUsedTimestamp(System.currentTimeMillis());
                    
                    //将连接添加到活跃连接列表中
                    state.activeConnections.add(conn);
                    
                    state.requestCount++;
                    
                    state.accumulatedRequestTime += System.currentTimeMillis() - t;
                //如果连接无效
                } else {
                   
                    if (log.isDebugEnabled()) {
                        log.debug("A bad connection (" 
                                  + conn.getRealHashCode() 
                                  + ") was returned from the pool"
                    }
                    //设置一些属性              
                    state.badConnectionCount++;
                    localBadConnectionCount++;
                    conn = null;
                    //如果失败太多就抛出异常              
                    if (localBadConnectionCount 
                        > (poolMaximumIdleConnections +
                           poolMaximumLocalBadConnectionTolerance)) {
                        
                        if (log.isDebugEnabled()) {
                            log.debug("PooledDataSource: Cbase.");
                        }
                        throw new SQLException("PooledDatase.");
                    }
                }
            }
        }
    }
    //while循环结束                             

    if (conn == null) {
        if (log.isDebugEnabled()) {
            log.debug("PooledDataSourection.");
        }
        throw new SQLException("PooledDattion.");
    }
	
    //返回连接                              
    return conn;
} 
```



#### 5.3.3 回收连接



然后我们来看一看回收连接的逻辑。回收连接成功与否只取决于空闲连接集合的状态，所需处理情况很少。

```java
//id:5.13
//PooledDataSource
protected void pushConnection(PooledConnection conn) throws SQLException {
	//加锁
    synchronized (state) {
        
        //把这个PooledConnection从活跃连接列表中移除
        state.activeConnections.remove(conn);
        
        //如果连接是合法的，则
        if (conn.isValid()) {
            
            //如果空闲连接列表未满
            if (state.idleConnections.size() < poolMaximumIdleConnections 
                && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
                
                //更新state中的状态
                state.accumulatedCheckoutTime += conn.getCheckoutTime();
                
                //回滚这个连接中还未提交的操作
                if (!conn.getRealConnection().getAutoCommit()) {
                    conn.getRealConnection().rollback();
                }
                
                //用老连接创建一个新的PooledConnection连接
                PooledConnection newConn 
                    = new PooledConnection(conn.getRealConnection(), this);
                
                //设置一些属性
                state.idleConnections.add(newConn);
                newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
                newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());
                
                //设置老连接无效
                conn.invalidate();
                
                if (log.isDebugEnabled()) {
                    log.debug("Returned connection " 
                              + newConn.getRealHashCode() + " to pool.");
                }
                
                //激活所有等待的线程
                state.notifyAll();
                
             //如果空闲连接列表已满   
            } else {
                
                //更新state中的状态
                state.accumulatedCheckoutTime += conn.getCheckoutTime();
                
                //回滚这个连接中还未提交的操作
                if (!conn.getRealConnection().getAutoCommit()) {
                    conn.getRealConnection().rollback();
                }
                
                //直接关闭这个连接
                conn.getRealConnection().close();
                
                if (log.isDebugEnabled()) {
                    log.debug("Closed connection " + conn.getRealHashCode() + ".");
                }
                
                //设置老连接无效
                conn.invalidate();
            }
            
         //如果连接是非法的，则   
        } else {
            if (log.isDebugEnabled()) {
                log.debug("A bad connecttion.");
            }
            state.badConnectionCount++;
        }
    }
}
```



上面代码首先将连接从活跃连接集合中移除，然后再根据空闲集合是否有空闲空间进行后续处理。如果空闲集合未满，此时复用原连接的字段信息创建新的连接，并将其放入空闲集合中即可。若空闲集合已满，此时无需回收连接，直接关闭即可。



我们知道获取连接的方法`popConnection`是由`getConnection`方法调用的，那回收连接的方法`pushConnection`是由谁调用的呢？答案是`PooledConnection`中的代理逻辑。我们来看看实际执行代理逻辑的`invoke()`方法都需要做什么。

```java
//5.14
//PooledConnection
//实际的代理逻辑
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    
    //获取当前调用的方法名
    String methodName = method.getName();
    
    //如果connection.close()被调用
    if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
        //并不会实际的去关闭这个连接，而是调用pushConnection()，分情况处理
      	dataSource.pushConnection(this);
      	return null;
    }
    //对于Connection中其他方法的调用
    try {
      //检查连接状态  
      if (!Object.class.equals(method.getDeclaringClass())) {
        // issue #579 toString() should never fail
        // throw an SQLException instead of a Runtime
        checkConnection();
      }
      //调用真实连接的对应方法  
      return method.invoke(realConnection, args);
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }

}
```

至此，回收连接的逻辑也看完了。