---
title: '[MyBatis源码][6][缓存机制]'
date: 2020-03-30 19:55:22
tags:
    - MyBatis
categories:
    - MyBatis
---
## 第六章 缓存机制



在 Web 应用中，缓存是必不可少的组件。通常我们都会用 Redis 或 memcached 等缓存中间件，拦截大量奔向数据库的请求，以减轻数据库压力。作为一个重要的组件，MyBatis自然也在内部提供了相应的支持。通过在框架层面增加缓存功能，可减轻数据库的压力，同时又可以提升查询速度，可谓一举两得。MyBatis 缓存结构由一级缓存和二级缓存构成，这两级缓存均是使用 `Cache` 接口的实现类。因此本章将首先会向大家介绍 `Cache` 几种实现类
的源码，然后再分析一级和二级缓存的实现。



### 6.1 缓存类介绍



在MyBatis中，Cache是缓存接口，它定义了一些基本的缓存操作，所有缓存类都应该实现该接口。实现类如下

1. 具有基本缓存功能的`PerpetualCache`
2. 具有`LRU`策略的缓存`LruCache`
3. 可保证线程安全的缓存`SynchronizedCache`
4. 具有阻塞功能的`BlockingCache`



此外，除了`PerpetualCache`外，其他的类全都是装饰器类。类图如下

![](https://i.loli.net/2020/03/30/mDMvY5KWBf6b2qG.png)

下面先看一下`Cache`接口

```java
//id:6.0
public interface Cache {

  /**
   * @return The identifier of this cache
   */
  //获取这个缓存的唯一标识符  
  String getId();

  /**
   * @param key Can be any object but usually it is a {@link CacheKey}
   * @param value The result of a select.
   */
  //将一个键值对存入缓存  
  void putObject(Object key, Object value);

  /**
   * @param key The key
   * @return The object stored in the cache.
   */
  //通过键从缓存中取出值  
  Object getObject(Object key);

  /**
   * As of 3.3.0 this method is only called during a rollback
   * for any previous value that was missing in the cache.
   * This lets any blocking cache to release the lock that
   * may have previously put on the key.
   * A blocking cache puts a lock when a value is null
   * and releases it when the value is back again.
   * This way other threads will wait for the value to be
   * available instead of hitting the database.
   *
   *
   * @param key The key
   * @return Not used
   */
  //删除给定键对应的键值对  
  Object removeObject(Object key);

  /**
   * Clears this cache instance.
   */
  //清空缓存  
  void clear();

  /**
   * Optional. This method is not called by the core.
   *
   * @return The number of elements stored in the cache (not its capacity).
   */
  //缓存一共存储了多少个键值对  
  int getSize();

  /**
   * Optional. As of 3.2.6 this method is no longer called by the core.
   * <p>
   * Any locking needed by the cache must be provided internally by the cache provider.
   *
   * @return A ReadWriteLock
   */
  //返回一个读写锁  
  ReadWriteLock getReadWriteLock();

}
```



#### 6.1.1 PerpetualCache

因为这个被修饰的`Cache`逻辑比较简单，这里只给出源代码

```java
////id:6.1
public class PerpetualCache implements Cache {

    private final String id;
  
    private Map<Object, Object> cache = new HashMap<>();
  
    public PerpetualCache(String id) {
        this.id = id;
    }
  
    @Override
    public String getId() {
        return id;
    }
  
    @Override
    public int getSize() {
        return cache.size();
    }
  
    @Override
    public void putObject(Object key, Object value) {
        cache.put(key, value);
    }
  
    @Override
    public Object getObject(Object key) {
        return cache.get(key);
    }
  
    @Override
    public Object removeObject(Object key) {
        return cache.remove(key);
    }
  
    @Override
    public void clear() {
        cache.clear();
    }
  
    @Override
    public ReadWriteLock getReadWriteLock() {
        return null;
    }
  
    @Override
    public boolean equals(Object o) {
        if (getId() == null) {
            throw new CacheException("Cache instances require an ID.");
        }
        if (this == o) {
            return true;
        }
        if (!(o instanceof Cache)) {
            return false;
        }
    
        Cache otherCache = (Cache) o;
        return getId().equals(otherCache.getId());
    }
  
    @Override
    public int hashCode() {
        if (getId() == null) {
            throw new CacheException("Cache instances require an ID.");
        }
        return getId().hashCode();
    }
  
}
```







#### 6.1.2 LruCache



先介绍`LRU`策略。`LRU`策略其实很简单。我们使用的缓存是一块给定大小的区域，它不是无限的。那么当缓存区域满了之后，我们再插入时，就需要选择一个老缓存删除它。对`LRU`策略来说，删除的是访问时间最早的那一条，因为它在近期是最少被访问的。因此，我们如果要实现`LRU`策略，就需要维护一个顺序表结构，表中元素按照访问时间先后进行排序，并且当表中某个元素被访问之后，它就要被放到表尾。当我们需要删除一个老缓存时，则直接删除表头的缓存即可。在MyBatis中，是使用`LinkedHashList`作为`LRU`结构的。



下面我们直接看一看`LruCache`的源代码

```java
//id:6.2
public class LruCache implements Cache {

  //被修饰的类
  private final Cache delegate;
  //它将被初始化为一个链式哈希表
  private Map<Object, Object> keyMap;
  //一个局部变量，用于执行删除
  private Object eldestKey;

  public LruCache(Cache delegate) {
    this.delegate = delegate;
    setSize(1024);
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    return delegate.getSize();
  }
  
  public void setSize(final int size) {
    //重新生成一个keyMap  
    keyMap = new LinkedHashMap<Object, Object>(size, .75F, true) {
      private static final long serialVersionUID = 4267176411845948333L;

      //重载了removeEldestEntry
      @Override
      protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
        boolean tooBig = size() > size;
        if (tooBig) {
          eldestKey = eldest.getKey();
        }
        return tooBig;
      }
    };
  }

  @Override
  public void putObject(Object key, Object value) {
    //先执行被装饰者的putObject  
    delegate.putObject(key, value);
    //然后维持LRU  
    cycleKeyList(key);
  }

  @Override
  public Object getObject(Object key) {
    //刷新key在keyMap中的位置以维持LRU
    keyMap.get(key); //touch
    return delegate.getObject(key);
  }
    
  
  @Override
  public Object removeObject(Object key) {
    return delegate.removeObject(key);
  }

  @Override
  public void clear() {
    //调用被修饰类的clear
    delegate.clear();
    //清空keyMap  
    keyMap.clear();
  }
    
  //获取读写锁，直接返回null	
  @Override
  public ReadWriteLock getReadWriteLock() {
    return null;
  }
    
  //维持keyMap中的LRU特性
  private void cycleKeyList(Object key) {
    //存储key到keyMap中  
    keyMap.put(key, key);
    if (eldestKey != null) {
      //从被装饰类中移除对应的缓存项  
      delegate.removeObject(eldestKey);
      eldestKey = null;
    }
  }
}
```



#### 6.1.3 BlockingCache

`BlockingCache` 实现了阻塞特性，该特性是基于 Java 重入锁实现的。同一时刻下，`BlockingCache` 仅允许一个线程访问指定 `key` 的缓存项，其他线程将会被阻塞住。下面我们来看一下 `BlockingCache` 的源码。



```java
//6.3
//BlockingCache
public class BlockingCache implements Cache {

    private long timeout;
    private final Cache delegate;
    private final ConcurrentHashMap<Object, ReentrantLock> locks;
  
    public BlockingCache(Cache delegate) {
        this.delegate = delegate;
        this.locks = new ConcurrentHashMap<>();
    }
  
    //放入值
    @Override
    public void putObject(Object key, Object value) {
        try {
            //存储缓存项
            delegate.putObject(key, value);
        } finally {
            //释放锁
            releaseLock(key);
        }
    }
  
    //读取值
    @Override
    public Object getObject(Object key) {
        //请求锁
        acquireLock(key);
        //读取值
        Object value = delegate.getObject(key);
        //若值不为空，释放锁
        if (value != null) {
            releaseLock(key);
        }
        return value;
    }
  
    //移除值
    @Override
    public Object removeObject(Object key) {
        //释放锁
        releaseLock(key);
        return null;
    }
  
    @Override
    public void clear() {
        delegate.clear();
    }
  
    @Override
    public ReadWriteLock getReadWriteLock() {
        return null;
    }
  
    //获取针对key的锁
    private ReentrantLock getLockForKey(Object key) {
        return locks.computeIfAbsent(key, k -> new ReentrantLock());
    }
  
    //请求锁
    private void acquireLock(Object key) {
        //从锁映射中获取锁
        Lock lock = getLockForKey(key);
        
        //如果超时大于0，则试着加锁
        if (timeout > 0) {
            
            try {
                boolean acquired = lock.tryLock(timeout, TimeUnit.MILLISECONDS);
                if (!acquired) {
                    throw new CacheException("Couldn't get a lock in " 
                                             + timeout 
                                             + " for the key " 
                                             +  key 
                                             + " at the cache " 
                                             + delegate.getId());
                }
            } catch (InterruptedException e) {
                throw new CacheException("Got interrupted while trying"
                                         + "to acquire lock for key " 
                                         + key, e);
            }
        //若超时等于0.直接加锁    
        } else {
            lock.lock();
        }
    }
  
    //释放锁
    private void releaseLock(Object key) {
        //从锁映射中获取锁
        ReentrantLock lock = locks.get(key);
        
        //如果锁被当前线程持有，则释放锁
        if (lock.isHeldByCurrentThread()) {
            lock.unlock();
        }
    }
  
}
```



大家看到`getObject`的实现可能会感觉有点奇怪。下面解释下，`getObject` 方法会先获取与 `key` 对应的锁，并加锁。若缓存命中，`getObject` 方法会释放锁，否则将一直锁定。`getObject` 方法若返回 `null`，表示缓存未命中。此时 `MyBatis` 会向数据库发起查询请求，并调用 `putObject` 方法存储查询结果。此时，`putObject` 方法会将指定 `key` 对应的锁进行解锁，这样被阻塞的线程即可恢复运行。



上面的解释对应了`BlockingCache`类注释中的下面这段话

> It sets a lock over a cache key when the element is not found in cache. This way, other threads will wait until this element is filled instead of hitting the database.



### 6.2 CacheKey



`CacheKey`类主要抽象了`Cache`中的`k-v`对中的`key`。因为影响`key`的因素不仅仅是SQL语句，还有运行时参数、分页参数等等因素。所以我们采用了`CacheKey`这个复合对象，来涵盖可影响查询结果的因子。



下面我们来看看源码

```java
//6.4
//CacheKey
public class CacheKey implements Cloneable, Serializable {
  
    //乘子，默认为37
    private final int multiplier;
    
    //CacheKey的hashcode，综合了各个影响因素
    private int hashcode;
    
    //校验和
    private long checksum;

    //影响因素的个数
    private int count;

    //影响因素集合
    private List<Object> updateList;
  
    
    public CacheKey() {
      this.hashcode = DEFAULT_HASHCODE;
      this.multiplier = DEFAULT_MULTIPLYER;
      this.count = 0;
      this.updateList = new ArrayList<>();
    }
}
```



上面4个变量，除了`multiplier`是恒定不变的之外，其他变量都会在更新操作中被修改。下面我们看一看更新操作的代码。每当修改发生时，我们就要一并更新`hash`值。

```java
//6.5
//CacheKey
public void update(Object object) {
    //计算这个新对象的hashCode作为baseHashCode
    int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);

    //自增count
    count++;
    
    //校验和加上bashHashCode
    checksum += baseHashCode;
    
    
    baseHashCode *= count;

    //更新hashCode，加入baseHashCode的影响
    hashcode = multiplier * hashcode + baseHashCode;

    //保存影响因子
    updateList.add(object);
  }
```



此外，由于`CacheKey`最终要作为键存入`HashMap`，因此它需要覆盖`equals`和`hashCode`方法。下面看看这两个方法的实现。

```java
//6.6
//CacheKey
@Override
public boolean equals(Object object) {
    
    //先直接比较这两个对象的地址是否相同，相同则是同一个对象，直接返回true
    if (this == object) {
        return true;
    }
    
    //看看要比较的类是不是CacheKey，不是直接返回false
    if (!(object instanceof CacheKey)) {
        return false;
    }

    final CacheKey cacheKey = (CacheKey) object;

    //比较两个cacheKey的hashcode是否相同
    if (hashcode != cacheKey.hashcode) {
        return false;
    }
    
    //比较两个cacheKey的校验和是否相同
    if (checksum != cacheKey.checksum) {
        return false;
    }

    //比较两个cacheKey的影响因素是否相同
    if (count != cacheKey.count) {
        return false;
    }

    //遍历每个影响因素，比较每个影响因素是否相同
    for (int i = 0; i < updateList.size(); i++) {
        Object thisObject = updateList.get(i);
        Object thatObject = cacheKey.updateList.get(i);
        if (!ArrayUtil.equals(thisObject, thatObject)) {
        return false;
        }
    }
    
    //都通过，则返回true
    return true;
}

@Override
public int hashCode() {
    return hashcode;
}
```





### 6.3 一级缓存



在进行数据库查询时，MyBatis总是按二级缓存、一级缓存、数据库的顺序进行。每个`SqlSession`都共享一个一级缓存。但一级缓存不可以跨`SqlSession`，因此不会存在并发问题。此外，一级缓存所存储的查询结果会在MyBatis执行更新操作、提交、回滚时被清空。它通常用于一次会话中，多次查询同一个查询的情况。



一级缓存是在`BaseExecutor`中被初始化的，这个缓存仅仅是一个`PerpetualCache`，没有任何装饰器。下面我们来看一看相关的初始化逻辑。

```java
//6.7
//BaseExecutor
//一级缓存的实例
protected PerpetualCache localCache;

protected BaseExecutor(Configuration configuration, Transaction transaction) {
	//...
    this.localCache = new PerpetualCache("LocalCache");
	//...
}
```



初始化非常简单，接着我们再看看访问一级缓存的逻辑

```java
//6.8
//BaseExecutor
public <E> List<E> query(MappedStatement ms,
                           Object parameter,
                           RowBounds rowBounds,
                           ResultHandler resultHandler) throws SQLException {
      
    //从ms上计算出BoundSql
    BoundSql boundSql = ms.getBoundSql(parameter);
    
    //根据多个影响因素创建一个CacheKey
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    
    //调用重载方法，将这个CacheKey也作为参数传入
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```



首先我们先从代码清单`6.8`的第12行`CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);`向下，看看如何通过这些影响因素创建一个CacheKey。

```java
//6.9
//BaseExecutor
public CacheKey createCacheKey(MappedStatement ms,
                               Object parameterObject,
                               RowBounds rowBounds,
                               BoundSql boundSql) {
    //检测这个执行器是否已经被关闭了
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    
    //新建一个CacheKey
    CacheKey cacheKey = new CacheKey();
    //将mappedStatement的id作为一个影响因素
    cacheKey.update(ms.getId());
    //将rouBound的limit和offset作为两个影响因素，它可以代表分页
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    
    //将SQL语句作为一个影响因素
    cacheKey.update(boundSql.getSql());
    
    //获取参数的形参列表
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    
    //后面这部分代码就是通过形参取出每个运行时的实参，然后每个实参都作为一个影响因素
    TypeHandlerRegistry typeHandlerRegistry 
        = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
        if (parameterMapping.getMode() != ParameterMode.OUT) {
            Object value;
            String propertyName = parameterMapping.getProperty();
            if (boundSql.hasAdditionalParameter(propertyName)) {
                value = boundSql.getAdditionalParameter(propertyName);
            } else if (parameterObject == null) {
                value = null;
            } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                value = parameterObject;
            } else {
                MetaObject metaObject = configuration.newMetaObject(parameterObject);
                value = metaObject.getValue(propertyName);
            }
            cacheKey.update(value);
        }
    }
    
    //将环境id也作为一个影响因素
    if (configuration.getEnvironment() != null) {
        // issue #176
        cacheKey.update(configuration.getEnvironment().getId());
    }
    
    return cacheKey;
}
```

通过上面的代码，我们可以看到：`CacheKey`中的影响因素有：`MappedStatement`的`id`、SQL语句、分页参数、运行时参数、`Environment`的`id`。



然后我们从代码清单`6.8`的15行`return query(ms, parameter, rowBounds, resultHandler, key, boundSql);`看看这个重载方法做了什么

```java
//6.10
//BaseExecutor
public <E> List<E> query(MappedStatement ms,
                         Object parameter,
                         RowBounds rowBounds,
                         ResultHandler resultHandler,
                         CacheKey key,
                         BoundSql boundSql) throws SQLException {
    
    List<E> list;
    
    try {
        queryStack++;
        
        //根据key从缓存中查询对应的结果
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        
        //如果查询到
        if (list != null) {
            //执行一个与存储过程相关的逻辑
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        
        //如果没有查询到    
        } else {
            
            //从数据库中查询
            list = queryFromDatabase(
                ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
        
    } finally {
        queryStack--;
    }
    
    return list;

}
```

上面的源码是精简版的，也就是先通过缓存查询。若缓存中没有，再调用`queryFromDatabase`查询数据库。接着我们再看看如何查询数据库。



```java
//6.11
//BaseExecutor
private <E> List<E> queryFromDatabase(MappedStatement ms,
                                      Object parameter,
                                      RowBounds rowBounds,
                                      ResultHandler resultHandler,
                                      CacheKey key,
                                      BoundSql boundSql) throws SQLException {
    
    List<E> list;
    
    //先在缓存中放一个占位符
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    
    //执行实际的查询逻辑
    try {
      	list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        //移除占位符
      	localCache.removeObject(key);
    }
    
    //将实际值存入缓存
    localCache.putObject(key, list);
    
    //执行存储过程相关的逻辑
    if (ms.getStatementType() == StatementType.CALLABLE) {
      	localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}
```

到此，一级缓存的逻辑就分析完了。因为它不用考虑并发问题，所以实现起来比较简单。下一节我们继续看二级缓存的实现。



### 6.4 二级缓存

```java
//6.12
//CachingExecutor
public <E> List<E> query(MappedStatement ms,
                         Object parameterObject,
                         RowBounds rowBounds,
                         ResultHandler resultHandler) throws SQLException {
    
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    //创建CacheKey
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    //调用重载方法，将创建的key也作为参数
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}

public <E> List<E> query(MappedStatement ms,
                         Object parameterObject,
                         RowBounds rowBounds,
                         ResultHandler resultHandler,
                         CacheKey key,
                         BoundSql boundSql) throws SQLException {
    
    //从这次SQL操作对应的MappedStatement中获取Cache
    Cache cache = ms.getCache();
    
    //如果配置文件中开启了二级缓存，那么这里可以获得这个cache
    if (cache != null) {
        
        flushCacheIfRequired(ms);
        
        if (ms.isUseCache() && resultHandler == null) {
            
            //确保没有输出参数
            ensureNoOutParams(ms, boundSql);
            
            //从二级缓存实例中通过这个key获取缓存中对应的值
            @SuppressWarnings("unchecked")
            List<E> list = (List<E>) tcm.getObject(cache, key);
            
            //如果缓存未命中
            if (list == null) {
                
                //调用被装饰者的query
                list = delegate.query(
                    ms, parameterObject, rowBounds, resultHandler, key, boundSql);
                
                //将查询到的结果放入缓存。
                tcm.putObject(cache, key, list); // issue #578 and #116
            }
            
            return list;
        }
    }
    
    return delegate.query(
        ms, parameterObject, rowBounds, resultHandler, key, boundSql);

}
```

由上文，`tcm`（`TransactionCacheManager`，事务缓存管理器）负责了查询缓存和向缓存中放入对象，我们接着看看它的逻辑。



`TransactionalCacheManager`内部维护了`Cache`实例与`TransactionalCache`实例间的映射关系，该类也仅负责维护两者的映射关系，真正做事的还是`TransactionalCache`。它是一个实现了`Cache`的装饰器，为被装饰的`Cache`实例增加事务功能。

```java
//6.12
//TransactionalCacheManager
public class TransactionalCacheManager {

    //一个从Cache到TransactionalCache的映射
    private final Map<Cache, TransactionalCache> transactionalCaches 
        = new HashMap<>();
  
    //清空指定的Cache
    public void clear(Cache cache) {
        getTransactionalCache(cache).clear();
    }
  
    //从TransactionnalCache获取值
    public Object getObject(Cache cache, CacheKey key) {
        return getTransactionalCache(cache).getObject(key);
    }
    
    //向TransactionnalCache放入值
    public void putObject(Cache cache, CacheKey key, Object value) {
        getTransactionalCache(cache).putObject(key, value);
    }
  
    //提交映射表中的所有TransactionalCache
    public void commit() {
        for (TransactionalCache txCache : transactionalCaches.values()) {
            txCache.commit();
        }
    }
  
    //回滚映射表中的所有TransactionalCache
    public void rollback() {
        for (TransactionalCache txCache : transactionalCaches.values()) {
            txCache.rollback();
        }
    }
  
    //通过Cache查找对应的TransactionalCache
    private TransactionalCache getTransactionalCache(Cache cache) {
        return transactionalCaches.computeIfAbsent(cache, TransactionalCache::new);
    }
  
}
```

xxx

```java
//6.13
//TransactionalCache
public class TransactionalCache implements Cache {

    private static final Log log = LogFactory.getLog(TransactionalCache.class);
  
    private final Cache delegate;
    private boolean clearOnCommit;
    //在事务被提交前，所有从数据库中查询的结果将缓存到此集合中
    private final Map<Object, Object> entriesToAddOnCommit;
    //在事务被提交前，当缓存未命中时，CacheKey将被存储到此集合中
    private final Set<Object> entriesMissedInCache;
  
    @Override
    public Object getObject(Object key) {
        
        //直接调用被装饰Cache的getobject
        Object object = delegate.getObject(key);
        
        //如果缓存未命中，就把这个key放入到entriesMissedInCache中
        if (object == null) {
            entriesMissedInCache.add(key);
        }
        
        // issue #146
        if (clearOnCommit) {
            return null;
        } else {
            return object;
        }
    }
  
	  
    @Override
    public void putObject(Object key, Object object) {
        //将需要缓存的键值对放入到entriesToAddOnCommit中
        entriesToAddOnCommit.put(key, object);
    }
  
    @Override
    public Object removeObject(Object key) {
        return null;
    }
  
    //
    @Override
    public void clear() {
        //将标志位clearOnCommit置为true
        clearOnCommit = true;
        //清空entriesToAddOnCommit
        entriesToAddOnCommit.clear();
    }
  
    public void commit() {
        //如果标志位clearOnCommit为true，则调用被装饰者的clear来清空
        if (clearOnCommit) {
            delegate.clear();
        }
        //刷新，将该存入的键值对存入到delegate中
        flushPendingEntries();
        //重置，将这个类的标志位、entriesToAddOnCommit、entriesMissedInCache全部clear
        reset();
    }
  
    //回滚
    public void rollback() {
        //
        unlockMissedEntries();
        ////重置，将这个类的标志位、entriesToAddOnCommit、entriesMissedInCache全部clear
        reset();
    }
  
    private void reset() {
        clearOnCommit = false;
        entriesToAddOnCommit.clear();
        entriesMissedInCache.clear();
    }
  
    private void flushPendingEntries() {
        //将需要放入缓存的键值对全部放入缓存
        for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
            delegate.putObject(entry.getKey(), entry.getValue());
        }
        
        //为所有刚才查询未查到的key存入空值
        for (Object entry : entriesMissedInCache) {
            if (!entriesToAddOnCommit.containsKey(entry)) {
            delegate.putObject(entry, null);
            }
        }
    }
  
   	//对entriesMissedInCache中的每个entry从被装饰者中删除掉
    private void unlockMissedEntries() {
        for (Object entry : entriesMissedInCache) {
            try {
            delegate.removeObject(entry);
            } catch (Exception e) {
            log.warn("Unexpected exception while notifiying a rollback to the cache adapter."
                + "Consider upgrading your cache adapter to the latest version.  Cause: " + e);
            }
        }
    }  
}
```



通过`TransactionalCache`我们可以解决数据库事务的脏读问题。



脏读问题的解决主要与`entriesToAddOnCommit`集合有关。该集合用于存储本次事务中查询的结果，那为什么要将结果保存在该集合中，而非 delegate 所表示的缓存中呢？主要是因为直接存到 delegate 会导致脏数据问题。



下面先看看，假如不采用这个集合来控制，为什么就会发生脏读。

![](https://i.loli.net/2020/03/30/eOzYlLq8tWQdhoU.png)

1. 时刻2，事务 A 对记录 A 进行了更新。
2. 时刻3，事务 A 从数据库查询记录A，并将记录 A 写入缓存中。
3. 时刻4，事务 B 查询记录 A，由于缓存中存在记录 A，事务B 直接从缓存中取数据。这个时候，脏数据问题就发生了。事务 B 在事务 A 未提交情况下，读取到了事务 A 所修改的记录。

这就是我们需要为每个事务引入一个独立缓存的原因：

- 查询数据时，仍从`delegate`缓存（以下统称为共享缓存）中查询。若缓存未命中，
  则查询数据库。
- 存储查询结果时，并不直接存储查询结果到共享缓存中，而是先存储到事务
  缓存中，也就是`entriesToAddOnCommit`集合。当事务提交时，再将事务缓存中的缓存项转
  存到共享缓存中。
- 这样，事务 B 只能在事务 A 提交后，才能读取到事务 A 所做的修改，解决了脏读问题。



下面我们举个例子说明脏读问题是如何得到解决的

![](https://i.loli.net/2020/03/30/kI4bCtyURJmzrTs.png)

1. 时刻2，事务 A 和 B 同时查询记录 A。此时共享缓存中还没没有数据，所以两个事务均会向数据库发起查询请求，并将查询结果存储到各自的事务缓存中。
2. 时刻3，事务 A 更新记录 A，这里把更新后的记录 A 记为 A′。
3. 时刻4，两个事务再次进行查询。此时，事务 A 读取到的记录为修改后的值，而事务 B 读取到的记录仍为原值。
4. 时刻5，事务 A被提交，并将事务缓存 A 中的内容转存到共享缓存中。
5. 时刻6，事务 B 再次查询记录 A，由于共享缓存中有相应的数据，所以直接取缓存数据即可。因此得到记录 A′，而非记录 A。但由于事务 A 已经提交，所以事务 B 读取到的记录 A′ 并非是脏数据。



但需要注意的时，MyBatis 缓存事务机制只能解决脏读问题，并**不能解决“不可重复读”问题**。再回到上图，事务 B 在被提交前进行了三次查询。前两次查询得到的结果为记录 A，最后一次查询得到的结果为 A′。最有一次的查询结果与前两次不同，这就会导致“不可重复读”的问题。MyBatis 的缓存事务机制最高只支持“读已提交”，并不能解决“不可重复读”问题。即使数据库使用了更高的隔离级别解决了这个问题，但因 MyBatis 缓存事务机制级别较低。此时仍然会导致“不可重复读”问题的发生，这个在日常开发中需要注意一下。
