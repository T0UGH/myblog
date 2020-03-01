---
title: '[Redis][9][数据库]'
date: 2020-03-01 16:29:33
tags:
    - redis
categories:
    - redis
---
## 第 9 章 数据库

本章将对Redis服务器的**数据库实现**进行详细介绍，说明**服务器保存数据库的方法**，**客户端切换数据库**的方法，**数据库保存键值对**的方法，以及针对**数据库的添加、删除、查看更新操作的实现方法**等。除此之外，本章还会说明**服务器保存键的过期时间**的方法，以及**服务器自动删除过期键**的方法，以及**数据库通知功能**的实现方法。

### 9.1 服务器中的数据库

Redis服务器将所有数据库都**保存**在服务器状态`redis.h/redisServer`结构的`db`数组中，`db`数组的每个项都是一个`redis.h/redisDb`结构，每个`redisDB`结构代表一个数据库：
````c
struct redisServer {
    //...

    //数据库
    redisDb *db;

    //服务器的数据库数量
    int dbnum;

    //...
}
````

dbnum属性决定了Redis**初始化**时创建数据库的**数量**，默认为16个，结构如下图

![](Redis-9-数据库\200301_0.png)


### 9.2 切换数据库

每个`Redis`客户端都有自己的**目标数据库**，每当客户端执行数据库操作的时候，目标数据库就会成为这些命令的**操作对象**。
**默认**情况下，`Redis`客户端的目标数据库为**0号数据库**，但客户端可以通过执行`SELECT`命令来**切换**目标数据库。

````sh
SELECT 2
````

在服务器内部，客户端状态`redisClient`结构的`db`属性记录了**客户端当前的目标数据库**，这个属性是一个指向`redisDB`结构的**指针**，被指向的元素就是客户端的目标数据库。

![](Redis-9-数据库\200301_1.png)

到目前为止，Redis仍然**没有**可以**返回客户端目标数据库**的命令。在数次切换数据库之后，你很可能会忘记自己当前正在使用的是哪个数据库。当出现这种情况时，为了避免对数据库进行误操作，在执行Redis命令特别是像`FLUSHDB`这样的**危险命令**之前，最好**先**执行一个`SELECT`命令，**显式切换**到指定的数据库，然后才执行别的命令。

### 9.3 数据库键空间

Redis是一个键值对（key-value pair）数据库服务器，`redis.h`结构的`dict`**字典**保存了数据库中的所有**键值对**，我们将这个字典称为**键空间**
- **键空间的键**也就是**数据库的键**，每个键都是一个字符串对象。
- **键空间的值**也就是**数据库的值**，每个值可以是字符串对象、列表对象、哈希表对象、集合对象和有序集合对象中的任意一种Redis对象。

![](Redis-9-数据库\200301_2.png)

#### 9.3.1 添加新键

添加一个新键值对到数据库，实际上就是将一个**新键值对**添加到**键空间**字典里面，其中键为字符串对象，而值则为任意一种类型的Redis对象。

#### 9.3.2 删除键

删除数据库中的一个键，实际上就是在**键空间**里面**删除**键所对应的键值对对象。

#### 9.3.3 更新键

略

#### 9.3.4 对键取值

略

#### 9.3.5 其他键空间操作

用于**清空**整个数据库的`FLUSHDB`命令，就是通过**删除**键空间中的所有**键值对**来实现的。

又比如说，用于**随机**返回数据库中某个键的 `RANDOMKEY`命令，就是通过在键空间中随机返回一个键来实现的。

#### 9.3.6 读写键空间时的维护操作

当使用Redis命令对数据库进行读写时，服务器不仅会对键空间执行指定的读写操作还会**执行一些额外的维护操作**，其中包括
- 在读取一个键之后，服务器会根据键**是否存在**来**更新**服务器的键空间**命中次数**或键空间不命中次数

- 在读取一个键之后，服务器会**更新键的LRU**（最后一次使用）时间，这个值可以用于计算键的闲置时间

- 如果服务器在读取一个键时发现该键已经**过期**，那么服务器会先**删除**这个过期键，然后才执行余下的其他操作

- 如果有客户端使用`WATCH`命令**监视**了某个键，那么服务器在对被监视的键进行**修改之后**，会将这个键标记为**脏**，从而让事务程序注意到这个键已经被修改过

### 9.4 设置键的生存时间或过期时间

客户端可以以秒或者毫秒精度为数据库中的某个**键**设置**生存时间**（Time To Live,TTL），在经过指定的秒数或者毫秒数之后，服务器就会**自动删除**生存时间为0的键。

#### 9.4.1 设置过期时间

Redis有四个不同的命令可以用于**设置**键的**生存时间**（键可以存在多久）或**过期时间**（键什么时候会被删除）
- `EXPIRE`：用于将键key的**生存时间**设置为xx**秒**。
- `PEXPIRE`：用于将键key的**生存时间**设置为xx**毫秒**。
- `EXPIREAT`：用于将键key的**过期时间**设置为指定的**秒数时间戳**
- `PEXPIREAT`：用于将键key的**过期时间**设置为指定的**毫秒数时间戳**

#### 9.4.2 保存过期时间

`redisDb`结构的`expires`字典**保存**了数据库中所有键的**过期时间**，我们称这个字典为**过期字典**：
- 过期字典的**键**是一个**指针**，这个指针**指向键空间**中的某个**键对象**
- 过期字典的**值**是一个`long long`类型的**整数**，这个整数**保存**了键所指向的数据库键的过期时间，它是一个毫秒精度的**UNIX时间戳**

例如下图
![](Redis-9-数据库\200301_3.png)

#### 9.4.3 移除过期时间

`PERSIST`命令可以**移除**一个键的过期时间

````sh
PERSIST message
````

#### 9.4.4 计算并返回剩余生存时间

`TTL`命令以**秒**为单位返回键的**剩余**生存时间，而`PTTL`命令则以**毫秒**为单位返回键的**剩余**生存时间：
````sh
TTL message
PTTL message
````

#### 9.4.5 过期键的判定

通过过期字典，程序可以用以下步骤检查一个给定键**是否过期**

1. 检查给定键**是否存在于过期字典**：如果存在，那么取得键的过期时间。
2. 检查当前UNIX**时间戳是否大于**键的过期时间：如果是的话，那么键已经过期；否则的话，键未过期。

### 9.5 过期键删除策略

过期键有三种删除策略
- **定时删除**：在设置键的过期时间的同时，创建一个**定时器**，让定时器在键的过期时间来临时，**立即执行**对键的删除操作。
- **惰性删除**：放任键过期不管，但是每次从键空间中**获取键**时，都**检査**取得的键是否**过期**，如果**过期**的话，就**删除**该键；如果没有过期，就返回该键。
- 定期删除：每**隔一段时间**，程序就对数据库进行一次**检査**，**删除**里面的**过期键**。至于要删除多少过期键，以及要检查多少个数据库，则由算法决定。

在这三种策略中，第一种和第三种为**主动删除**策略，而第二种则为**被动删除**策略。

#### 9.5.1 定时删除

定时删除策略对**内存**是最**友好**的：通过使用定时器，定时删除策略可以保证过期键会尽可能快地被删除，并释放过期键所占用的内存。

它对**CPU时间**是最**不友好**的：在过期键比较多的情况下，删除过期键这一行为可能会占用相当一部分CPU时间，

#### 9.5.2 惰性删除

惰性删除策略对**CPU**时间来说是最**友好**的：程序只会在取岀键时才对键进行过期检査这可以保证删除过期键的操作只会在**非做不可的情况下**进行，并且删除的**目标仅限于当前处理的键**，这个策略不会在删除其他无关的过期键上花费任何CPU时间。

惰性删除策略的缺点是，它对**内存**是最**不友好**的：如果一个键已经过期，而这个键又仍然保留在数据库中，那么只要这个过期键**不被访问**，它所占用的**内存永远不会释放**。

#### 9.5.3 定期删除

定期删除策略是前两种策略的一种**整合和折中**：
- 定期删除策略每**隔一段时间**执行一次**删除**过期键操作，并通过**限制**删除操作执行的**时长**和**频率**来减少删除操作对CPU时间的影响。
- 除此之外，通过定期删除过期键，定期删除策略有效地**减少**了因为过期键而带来的**内存浪费**。

### 9.6 Redis的过期删除策略

Redis服务器实际使用的是**惰性删除**和**定期删除**两种策略

#### 9.6.1 惰性删除策略的实现

**惰性删除**策略由`db.c/expireIfNeeded`函数实现，所有**读写数据库**的Redis命令在**执行之前**都会调用`expireIfNeeded`函数对**输入键**进行**检查**：
- 如果输入键已经**过期**，那么`expireIfNeeded`函数将输入键从数据库中**删除**
- 如果输入键**未过期**，那么`expireIfNeeded`函数**不做动作**。

流程图如下

![](Redis-9-数据库\200301_4.png)

![](Redis-9-数据库\200301_5.png)

`expireIfNeeded`源码如下
````c
/*
 * 检查 key 是否已经过期，如果是的话，将它从数据库中删除。
 *
 * 返回 0 表示键没有过期时间，或者键未过期。
 *
 * 返回 1 表示键已经因为过期而被删除了。
 */
int expireIfNeeded(redisDb *db, robj *key) {

    // 取出键的过期时间
    mstime_t when = getExpire(db,key);
    mstime_t now;

    // 没有过期时间
    if (when < 0) return 0; /* No expire for this key */

    /* Don't expire anything while loading. It will be done later. */
    // 如果服务器正在进行载入，那么不进行任何过期检查
    if (server.loading) return 0;

    /* If we are in the context of a Lua script, we claim that time is
     * blocked to when the Lua script started. This way a key can expire
     * only the first time it is accessed and not in the middle of the
     * script execution, making propagation to slaves / AOF consistent.
     * See issue #1525 on Github for more information. */
    now = server.lua_caller ? server.lua_time_start : mstime();

    /* If we are running in the context of a slave, return ASAP:
     * the slave key expiration is controlled by the master that will
     * send us synthesized DEL operations for expired keys.
     *
     * Still we try to return the right information to the caller, 
     * that is, 0 if we think the key should be still valid, 1 if
     * we think the key is expired at this time. */
    // 当服务器运行在 replication 模式时
    // 附属节点并不主动删除 key
    // 它只返回一个逻辑上正确的返回值
    // 真正的删除操作要等待主节点发来删除命令时才执行
    // 从而保证数据的同步
    if (server.masterhost != NULL) return now > when;

    // 运行到这里，表示键带有过期时间，并且服务器为主节点

    /* Return when this key has not expired */
    // 如果未过期，返回 0
    if (now <= when) return 0;

    /* Delete the key */
    server.stat_expiredkeys++;

    // 向 AOF 文件和附属节点传播过期信息
    propagateExpire(db,key);

    // 发送事件通知
    notifyKeyspaceEvent(REDIS_NOTIFY_EXPIRED,
        "expired",key,db->id);

    // 将过期键从数据库中删除
    return dbDelete(db,key);
}
````

#### 9.6.2 定期删除策略的实现

过期键的**定期删除策略**由`redis.c/activeExpireCycle`函数实现，每当Redis的**服务器周期性**操作`redis.c/serverCron`函数执行时，`activeExpireCycle`函数就会**被调用**，它在**规定的时间**内，分**多次遍历**服务器中的各个数据库，从数据库的`expires`字典中**随机检查**一部分键的过期时间，并**删除**其中的过期键。

````py
# 默认每次检查的数据库数量
DEFAULT_DB_NUMBERS = 16

# 默认每个数据库检查的键数量
DEFAULT_KEY_NUMBERS = 2

# 全局变量，记录检查进度到哪个数据库
current_db = 0

def activeExpireCycle():
    # 初始化要检查的数据库数量
    # 如果服务器的数据库数量比 DEFAULT_DB_NUMBERS要小那么以服务器的数据库数量为准
    if server.donum < DEFAULT_DB_NUMBERS:
        db_numbers = server.dbnum 
    else:
        db_numbers = DEFAULT_DB_NUMBERS
        
    # 遍历各个数据库
    for i in range（db numbers）

        # 如果current_db的值等于服务器的数据库数量
        # 这表示检查程序已经遍历了服务器的所有数据库一次
        #将current_db重置为0，开始新的一轮遍历
        if current_db = server.dbnum current_db= 0
        
        # 获取当前要处理的数据库
        redisDB = server.db[current db]

        # 将数据库索引增1，指向下一个要处理的数据库
        current_db += 1
        # 检查数据库键
        for j in range（DEFAULT KEY NUMBERS）：
            # 如果数据库中没有一个键带有过期时间，那么跳过这个数据库
            if redisDb.expires.size()==0： 
                break

            # 随机获取一个带有过期时间的键
            key_with_ttl = redisDb.expires.get_random_key()

            # 检查键是否过期，如果过期就删除它
            if is_expired(key_with_ttl):
                delete_key(key_with_ttl)
                
            # 已达到时间上限，停止处理
            if reach_time_limit():
                return
````
`activeExpireCycle`函数的**工作模式**可以总结为
- 函数每次运行时，都从一定数量的数据库中取出一定数量的**随机键**进行**检查**，并**删除**其中的**过期键**。
- 全局变量`current_db`会**记录**当前`activeExpireCycle`函数检查的**进度**，并在下一次`activeExpireCycle`函数调用时，**接着上一次的进度进行处理**
- 服务器中的**所有数据库**都会**被检查**一遍，这时函数将`current_db`变量**重置为0**，

### 9.7 AOP、RDB和复制功能对过期键的处理

在这一节，我们将探讨**过期键**对Redis服务器中**其他模块**的**影响**

#### 9.7.1 生成RDB文件

在执行`SAVE`命令或者`BGSAVE`命令**创建**一个新的**RDB文件**时，程序会对数据库中的**键**进行**检查**，已**过期的键**将**不保存**到新创建的RDB文件中。

#### 9.7.2 载入RDB文件

在启动Redis服务器时，如果服务器开启了RDB功能，那么服务器将对**RDB文件**进行**载入**
- 如果服务器以**主服务器模式**运行，那么在载入RDB文件时，程序会对文件中保存的键进行检査，未过期的键会被载入到数据库中，而**过期键**则会被**忽略**，
- 如果服务器以**从服务器模式**运行，那么在载入RDB文件时，文件中保存的所有键，**不论是否过期**，**都会**被**载入**到数据库中。不过，因为主从服务器在进行**数据同步**的时候，**从服务器**的数据库就会被**清空**，所以一般来讲，过期键对载入RDB文件的从服务器也**不会造成影响**。

#### 9.7.3 AOF文件写入

当服务器以**AOF持久化模式**运行时，如果数据库中的某个**键**已经**过期**，但它还**没有**被惰性删除或者定期**删除**，那么AOF文件**不会**因为这个过期键而**产生**任何**影响**。
当过期键**被**惰性**删除**或者定期删除之后，程序会向AOF文件**追加**（append）一条**DEL命令**，来显式地记录该键已被删除。

#### 9.7.4 AOF重写

在执行AOF重写的过程中，程序会对数据库中的键进行检查，**已过期**的键**不会**被**保存**到重写后的AOF文件中。

#### 9.7.5 复制

当服务器运行在复制模式下时，从服务器的过期键**删除**动作由**主服务器控制**：
- **主服务器**在**删除**一个过期键之**后**，会显式地向所有**从服务器**发送一个**DEL命令**，**告知从服务器删除**这个过期键。
- **从服务器**在**执行客户端**发送的读**命令**时，即使碰到过期键也**不会将过期键删除**，而是继续像处理未过期的键一样来处理过期键。
- 从服务器**只有**在**接到主服务器**发来的DEL**命令**之后，才会**删除**过期键。

具体流程如下
![](Redis-9-数据库\200301_6.png)
![](Redis-9-数据库\200301_7.png)
![](Redis-9-数据库\200301_8.png)
![](Redis-9-数据库\200301_9.png)

### 9.8 数据库通知

**数据库通知**功能可以让客户端通过**订阅**给定的频道或者模式，来**获知**数据库中**键的变化**，以及数据库中**命令的执行情况**。

其中
- 关注“某个键执行了什么命令”的通知称为**键空间通知**
- 关注“某个命令被什么键执行了”的通知称为**键事件通知**

#### 9.8.1 发送消息

发送数据库通知的功能是由`notify.c/notifykeyspaceEvent`函数实现的：

每当一个Redis**命令**需要**发送**数据库**通知**的时候，该命令的实现函数就会**调用**`notifyKeyspaceEvent`函数，并向函数传递该命令所引发的事件的相关信息。

例如下面是`SADD`命令的实现函数`saddCommand`的一部分代码
````c
void saddCommand(redisClient *c){
    //...

    //如果至少有一个元素被成功添加，则执行
    if(added){
        //...

        //发送事件通知
        notifyKeyspaceEvent(REDIS_NOTIFY_SET, "sadd", c->argv[1], c->db->id)
    }
}
````

#### 9.8.2 发送通知的实现

下面是`notifyKeyspaceEvent`函数的伪代码实现
````py
def notifykeyspaceEvent(type, event, key, dbid):
    
    # 如果给定的通知不是服务器允许发送的通知，那么直接返回
    if not (server.notify_keyspace_events & type):
        return
    
    # 发送键空间通知
    if server.notify_keyspace_events & REDIS_NOTIFY_KEYSPACE:

        # 将通知发送给频道"__keyspace@{dbid}__:{key}
        # 内容为键所发生的事件<event>
        
        # 构建频道名字
        chan = "__keyspace@{dbid}__:{key}". format(dbid=dbid, key=key)

        # 发送通知
        pubsubPublishMessage(chan,event)

    # 发送键事件通知
    if server.notify_keyspace_events & REDIS_NOTIFY_KEYEVENT:
        
        # 将通知发送给频道"__keyevent@{dbid}__:{event}
        # 内容为发生事件的键<key>

        # 构建频道名字
        chan = "__keyspace@{dbid}__:{event}". format(dbid=dbid, event=event)
        
        # 发送通知
        pubsubPublishMessage(chan, key)
