---
title: '[Redis][13][客户端]'
date: 2020-03-04 16:06:57
tags:
    - redis
categories:
    - redis
---
## 第 13 章 客户端

Redis服务器是典型的**一对多服务器**程序：一个服务器可以与多个客户端建立网络连接。

通过使用由**I/O多路复用**技术实现的文件事件处理器，Redis服务器使用**单线程单进程**的方式来处理命令请求，并与多个客户端进行网络通信。

对于每个与服务器进行连接的客户端，服务器都为这些客户端建立了相应的`redis.h/redisclient`结构（客户端状态），这个结构保存了客户端当前的**状态信息**，以及**执行相关功能**时需要用到的**数据结构**，具体属性本节稍后会介绍

Redis服务器结构中存储了一个`redisClient`类型的链表，它保存了所有与服务器连接的客户端状态结构

### 13.1 客户端属性

`redisClient`结构的源码如下
````c
typedef struct redisClient {

    // 套接字描述符
    int fd;

    // 当前正在使用的数据库
    redisDb *db;

    // 当前正在使用的数据库的 id （号码）
    int dictid;

    // 客户端的名字
    robj *name;             /* As set by CLIENT SETNAME */

    // 查询缓冲区
    sds querybuf;

    // 查询缓冲区长度峰值
    size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size */

    // 参数数量
    int argc;

    // 参数对象数组
    robj **argv;

    // 记录被客户端执行的命令
    struct redisCommand *cmd, *lastcmd;

    // 请求的类型：内联命令还是多条命令
    int reqtype;

    // 剩余未读取的命令内容数量
    int multibulklen;       /* number of multi bulk arguments left to read */

    // 命令内容的长度
    long bulklen;           /* length of bulk argument in multi bulk request */

    // 回复链表
    list *reply;

    // 回复链表中对象的总大小
    unsigned long reply_bytes; /* Tot bytes of objects in reply list */

    // 已发送字节，处理 short write 用
    int sentlen;            /* Amount of bytes already sent in the current
                               buffer or object being sent. */

    // 创建客户端的时间
    time_t ctime;           /* Client creation time */

    // 客户端最后一次和服务器互动的时间
    time_t lastinteraction; /* time of the last interaction, used for timeout */

    // 客户端的输出缓冲区超过软性限制的时间
    time_t obuf_soft_limit_reached_time;

    // 客户端状态标志
    int flags;              /* REDIS_SLAVE | REDIS_MONITOR | REDIS_MULTI ... */

    // 当 server.requirepass 不为 NULL 时
    // 代表认证的状态
    // 0 代表未认证， 1 代表已认证
    int authenticated;      /* when requirepass is non-NULL */

    // 复制状态
    int replstate;          /* replication state if this is a slave */
    // 用于保存主服务器传来的 RDB 文件的文件描述符
    int repldbfd;           /* replication DB file descriptor */

    // 读取主服务器传来的 RDB 文件的偏移量
    off_t repldboff;        /* replication DB file offset */
    // 主服务器传来的 RDB 文件的大小
    off_t repldbsize;       /* replication DB file size */
    
    sds replpreamble;       /* replication DB preamble. */

    // 主服务器的复制偏移量
    long long reploff;      /* replication offset if this is our master */
    // 从服务器最后一次发送 REPLCONF ACK 时的偏移量
    long long repl_ack_off; /* replication ack offset, if this is a slave */
    // 从服务器最后一次发送 REPLCONF ACK 的时间
    long long repl_ack_time;/* replication ack time, if this is a slave */
    // 主服务器的 master run ID
    // 保存在客户端，用于执行部分重同步
    char replrunid[REDIS_RUN_ID_SIZE+1]; /* master run id if this is a master */
    // 从服务器的监听端口号
    int slave_listening_port; /* As configured with: SLAVECONF listening-port */

    // 事务状态
    multiState mstate;      /* MULTI/EXEC state */

    // 阻塞类型
    int btype;              /* Type of blocking op if REDIS_BLOCKED. */
    // 阻塞状态
    blockingState bpop;     /* blocking state */

    // 最后被写入的全局复制偏移量
    long long woff;         /* Last write global replication offset. */

    // 被监视的键
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */

    // 这个字典记录了客户端所有订阅的频道
    // 键为频道名字，值为 NULL
    // 也即是，一个频道的集合
    dict *pubsub_channels;  /* channels a client is interested in (SUBSCRIBE) */

    // 链表，包含多个 pubsubPattern 结构
    // 记录了所有订阅频道的客户端的信息
    // 新 pubsubPattern 结构总是被添加到表尾
    list *pubsub_patterns;  /* patterns a client is interested in (SUBSCRIBE) */
    sds peerid;             /* Cached peer ID. */

    /* Response buffer */
    // 回复偏移量
    int bufpos;
    // 回复缓冲区
    char buf[REDIS_REPLY_CHUNK_BYTES];

} redisClient;
````

#### 13.1.1 套接字描述符

客户端状态的`fd`属性记录了客户端正在使用的套接字描述符

若`fd = -1`则说明这个客户端是用来执行Lua脚本或者AOF操作的伪客户端

可以通过`CLIENT list`命令列出目前所有连接到服务器的普通客户端
````sh
CLIENT list
addr=127.0.0.1:53428 fd=6 name= age=1242 idle=0
addr=127.0.0.1:53428 fd=7 name= age=1242 idle=0
````

#### 13.1.2 名字

使用`CLIENT setname`命令可以为客户端设置名字，让客户端身份更加清晰

#### 13.1.3 标志

客户端的标记属性`flags`记录了客户端的**角色**和当前所处的**状态**

`flags`属性的值可以是多个标志的二进制或，例如
````c
flags = <flag1> | <flag2> | ...
flags = REDIS_LUA_CLIENT | REDIS_FORCE_AOF | REDIS_FORCE_REPL
````

一部分标志记录了客户端的角色

- 在主从服务器进行复制操作时，主服务器会成为从服务器的客户端，而从服务器也会成为主服务器的客户端。`REDIS_MASTER`标志表示客户端代表的是一个主服务器，`REDIS SLAVE`标志表示客户端代表的是一个从服务器。

- `REDIS_PRE_PSYNC`标志表示客户端代表的是一个版本低于Redis2.8的从服务器

- `REDIS_LUA_CLIENT`标识表示客户端是专门用于处理Lua脚本里面包含的Redis命令的伪客户端。

另外一部分标志记录了客户端目前所处的状态
- `REDIS_MONITOR`标志表示客户端正在执行`MONITOR`命令。
- `REDIS_UNIX_SOCKET`标志表示服务器使用`UNIX`套接字来连接客户端。
- `REDIS_BLOCKED`标志表示客户端正在被`BRPOP`、`BLPOP`等命令阻塞。
- 等等

#### 13.1.4 输入缓冲区

输入缓冲区是`querybuf`属性，它是一个Redis字符串结构，用于保存客户端发送的命令请求，大小可以根据输入内容动态地扩大和缩小。例子如下图

![](Redis-13-客户端\200304_0.png)

#### 13.1.5 命令与命令参数

在服务器将客户端发送的命令请求保存到客户端状态的querybuf属性之后，服务器将对命令请求的内容进行分析，并将得到的命令参数以及命令参数的个数分别存储在客户端状态的`argv`和`argc`中。

例如下图，假如客户端发送的命令为`SET key "value"`，那么将这样保存
![](Redis-13-客户端\200304_1.png)

#### 13.1.6 命令的实现函数

当服务器从协议内容中分析并得出`argv`属性和`argc`属性的值之后，服务器将根据项`argv[0]`的值，在命令表中查找命令所对应的命令实现函数。

命令表是一个字典，字典的键是一个SDS结构(Redis定义的字符串结构)，保存了命令的名字，而字典的值是命令所对应的redisCommand结构，这个结构保存了命令的实现函数、命令的标志、命令应该给定的参数个数等信息。

当程序成功的找到了`argv[0]`对应的`redisCommand`结构时，它会将客户端状态(`redisClient`)中的`cmd`指针指向这个命令

例如下图，我们在命令表中查找`SET`命令，并将`cmd`指针指向它
![](Redis-13-客户端\200304_2.png)

#### 13.1.7 输出缓冲区

执行命令得到的命令回复将保存在客户端状态的输出缓冲区中，它有定长和变长两种选择

#### 13.1.8 身份验证

略

#### 13.1.9 时间

略

### 13.2 客户端的创建与关闭

服务器使用不同的方式来创建和关闭不同类型的客户端

#### 13.2.1 创建普通客户端

如果客户端是通过网络连接与服务器进行连接的普通客户端，那么在客户端使用`connect`函数连接到服务器时，服务器就会调用连接事件处理器，为客户端创建相应的客户端状态，并将这个新的客户端状态添加到服务器状态结构`client`链表的尾部

如下图所示，当客户端c3连接时
![](Redis-13-客户端\200304_3.png)

#### 13.2.2 关闭普通客户端

一个普通客户端可以因为多种原因而关闭
- 客户端进程退出或者被杀死
- 客户端向服务器发送了不符合协议格式的数据
- 如果客户端成为`CLIENT KILL`命令的目标，它也会被关闭
- 如果设置了timeout配置项，当客户端的空转时间超过了timeout时就会被关闭
- 如果客户端发送的命令请求超过了输入缓冲区的限制大小，则它会被关闭
- 如果客户端发送的命令请求超过了输出缓冲区的限制大小，则它会被关闭

#### 13.2.3 Lua脚本的伪客户端

服务器在初始化时创建负责执行Lua脚本的伪客户端，并将这个伪客户端关联在服务器状态结构(`redisServer`)的`lua_client`属性中
````c
typedef struct redisServer{
    //...

    redisClient *lua_client
    //...
}redisServer
````

`lua_client`伪客户端在服务器运行的整个生命周期中都会存在

#### 13.2.4 AOF文件的伪客户端

服务器在**载入**AOF文件时，会创建用于执行存储于AOF中的**写命令**的伪客户端，当载入完成时，这个伪客户端就会关闭。