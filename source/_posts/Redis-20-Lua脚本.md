---
title: '[Redis][20][Lua脚本]'
date: 2020-03-08 18:37:42
tags:
    - redis
categories:
    - redis
---
## 第 20 章 Lua脚本

通过在服务器中嵌入Lua环境，Redis客户端可以使用Lua脚本，直接在服务器端原子地执行多个Redis命令。
- 使用EVAL命令可以直接对输入的脚本进行求值
- 使用EVALSHA命令则可以根据脚本的SHA1校验和来对脚本进行求值
- 使用SCRIPT LOAD命令可以载入这个校验和对应的脚本

本章的安排
1. 介绍Redis服务器初始化Lua环境的整个过程
2. 介绍与Lua环境进行协作的两个组件，它们分别是负责执行Lua脚本中包含的Redis命令的伪客户端以及负责保存传入服务器的Lua脚本的脚本字典
3. 介绍EVAL和EVALSHA命令的实现原理
4. 介绍管理脚本的四个命令：SCRIPT FLUSH命令、SCRIPT EXISTS命令、SCRIPT LOAD命令和SCRIPT KILL命令的实现原理
5. 介绍Redis在主从服务器之间复制Lua脚本的方法

### 20.1 创建并修改Lua环境

Redis在服务器内嵌了一个Lua环境并对这个Lua环境进行了一系列修改，

#### 20.1.1 创建Lua环境

在最开始的这一步，服务器首先调用Lua的`C API`函数`lua_open()`，创建一个新的Lua环境

#### 20.1.2 载入函数库

Redis修改Lua环境的第一步，就是将以下函数库载入到Lua环境里面：
- 基础库
- 表格库
- 字符串库
- 数学库
- 调试库
- Lua CJSON库
- Struct库
- Lua cmsgpack库

通过使用这些功能强大的函数库，Lua脚本可以直接对执行Redis命令获得的数据进行复杂的操作。

#### 20.1.3 创建Redis全局表格

在这一步，服务器将在Lua环境中创建一个Redis表格，并将它设为全局变量。这个Redis表格主要包含各种对Redis中的数据进行操作的函数，使得Lua脚本具有操作Redis数据库的能力
- 用于执行Redis命令的`redis.call`和`redis.pca11`函数。
- 用于记录Redis日志的`redis.1og`函数
- 用于计算SHA1校验和的`redis.sha1hex`函数。
- 用于返回错误信息的`redis.error_rep1y`函数和`redis.status_reply`函数。

在这些函数里面，最常用也最重要的要数`redis.ca11`函数和`redis.pca11`函数，通过这两个函数，用户可以直接在Lua脚本中执行Redis命令

````sh
redis> EVAL "return redis.call('PING')" 0
PONG
````

#### 20.1.4 使用Redis自制的随机函数来替换Lua原有的随机函数

为了保证**相同的脚本**可以在**不同的机器**上产生**相同的结果**， Redis要求所有传入服务器的Lua脚本，以及Lua环境中的所有函数，都必须是无副作用的纯函数。

因为这个原因，Redis使用**自制的函数**替换了`math`库中原有的`math.random`函数和`math.randomseed`函数，替换之后的两个函数有以下特征
- 对于相同的`seed`来说，`math.random`总产生相同的随机数序列，这个函数是纯函数。
- 除非在脚本中使用`math.randomseed`显式地修改`seed`，否则每次运行脚本时Lua环境都使用固定的`math.randomseed(0)`语句来初始化`seed`。

#### 20.1.5 创建排序辅助函数

对于一个集合键来说，因为集合元素的排列是无序的，所以即使两个集合的元素完全相同，它们的输出结果也可能并不相同。

为了消除这些命令带来的不确定性，服务器创建一个排序辅助函数`redis_compare_helper`，当Lua脚本执行完一个带有不确定性的命令之后，程序会使用`redis_compare_helper`作为对比函数，自动调用`table.sort`函数对命令的返回值做一次排序，以此来保证相同的数据集总是产生相同的输出。

#### 20.1.6 创建redis.pcall函数的错误报告辅助函数

在这一步，服务器将为Lua环境创建一个名为`redis_err_handler`的错误处理函数，当脚本调用`redis.pcall`函数执行Redis命令，并且被执行的命令出现错误时`redis_err_handler`就会打印出错代码的来源和发生错误的行数，为程序的调试提供方便。

#### 20.1.7 保护Lua的全局环境

在这一步，服务器将对Lua环境中的全局环境进行保护，确保传人服务器的脚本不会因为忘记使用1oca1关键字而将额外的全局变量添加到Lua环境里面，当一个脚本试图创建一个全局变量时，服务器将报告一个错误

#### 20.1.8 将Lua环境保存到服务器状态的lua属性里面


在最后的这一步，服务器会将Lua环境和服务器状态的lua属性关联起来，如下图所示。

![](Redis-20-Lua脚本\200308_4.png)

因为Redis使用串行化的方式来执行Redis命令，所以在任何特定时间里，最多都只会有一个脚本能够被放进Lua环境里面运行，因此，整个Redis服务器只需要创建一个Lua环境即可。

### 20.2 Lua环境协作组件

Redis还创建了与Lua环境进行协作的两个组件，它们分别是负责执行Lua脚本中包含的Redis命令的伪客户端以及负责保存传入服务器的Lua脚本的脚本字典

#### 20.2.1 伪客户端

因为执行Reds命令必须有相应的客户端状态，所以为了执行Lua脚本中包含的 Redis命令， Redis服务器专门为Lua环境创建了一个伪客户端

下图展示了Lua脚本在调用redis.call函数时，Lua环境、伪客户端、命令执行器之间的通信过程

![](Redis-20-Lua脚本\200308_5.png)

#### 20.2.2 lua_script字典

 Redis服务器为Lua环境创建的另一个协作组件是`lua_scripts`字典，这个字典的键为某个Lua脚本的SHA1校验和，而字典的值则是SHA1校验和对应的Lua脚本，例子如下图

![](Redis-20-Lua脚本\200308_6.png)

### 20.3 EVAL命令的实现

EVAL命令的执行过程可以分为以下三个步骤：
1. 根据客户端给定的Lua脚本，在Lua环境中定义一个Lua函数
2. 将客户端给定的脚本保存到`lua_scripts`字典，等待将来进一步使用。
3. 执行刚刚在Lua环境中定义的函数，以此来执行客户端给定的Lua脚本。

#### 20.3.1 定义脚本函数

当客户端向服务器发送EVAL命令，要求执行某个Lua脚本的时候，服务器首先要做的就是在Lua环境中，为传入的脚本定义一个与这个脚本相对应的Lua函数，其中，Lua函数的名字由f前缀加上脚本的SHA1校验和组成，而函数的体则是脚本本身。

例如
````lua
function f_5332031c6b470dc5a0dd9b4bf2030dea6d65de91()
    return 'hello, world'
end
````

使用函数来保存客户端传入的脚本有以下好处：
- 执行脚本的步骤非常简单，只要调用与脚本相对应的函数即可。
- 通过函数的局部性来让Lua环境保持清洁，减少了垃圾回收的工作量，并且避免了使用全局变量。
- 只要记得这个脚本的SHA1校验和，服务器就可以直接通过调用Lua函数来执行脚本，这是EVALSHA命令的实现原理

#### 20.3.2 将脚本保存到lua_script字典

EVAL命令要做的第二件事是将客户端传人的脚本保存到服务器的`1ua_scripts`字典里面。

#### 20.3.3 执行脚本函数

整个执行步骤如下
1. 将`EVAL`命令中传人的键名参数和脚本参数分别作为全局变量传入到Lua环境里面。
2. 为Lua环境装载超时处理钩子
3. 执行脚本函数
4. 移除之前装载的超时钩子
5. 将执行脚本函数所得的结果保存到客户端状态的输出缓冲区里面，等待服务器将结果返回给客户端。
6. 对Lua环境执行垃圾回收操作。

### 20.4 EVALSHA命令的实现

每个被EVAL命令成功执行过的Lua脚本，在Lua环境里面都有一个与这个脚本相对应的Lua函数，函数的名字由f前缀加上40个字符长的SHA1校验和组成，例如`f_5332031c6b470dc5a0dd9b4bf2030dea6d65de91`。

只要脚本对应的函数曾经在Lua环境里面定义过，那么即使不知道脚本的内容本身，客户端也可以根据脚本的SHA1校验和来调用脚本对应的函数，从而达到执行脚本的目的，这就是EVALSHA命令的实现原理。

伪代码如下
````py
def EVALSHA(sha1):

    # 拼接出函数的名字
    func_name = "f_" + sha1

    # 查看这个函数在Lua环境中是否存在
    if function_exists_in_lua_env(func_name):

        # 如果函数存在，那么执行它
        execute_lua_function(func_name)
    else:

        # 否则，返回一个错误信息
        send_script_error("SCRIPT NOT FOUND")
````

### 20.5 脚本管理命令的实现

#### 20.5.1 SCRIPT FLUSH

SCRIPT FLUSH命令用于清除服务器中所有和Lua脚本有关的信息，这个命令会释放并重建`lua_scripts`字典，关闭现有的Lua环境并重新创建一个新的Lua环境。

伪代码如下
````py
def SCRIPT_FLUSH():

    # 释放脚本字典
    dictRelease(server.lua_scripts)

    # 重建脚本字典
    server.lua_script = dictCreate()

    # 关闭Lua环境
    lua_close(server.lua)

    # 初始化一个新的Lua环境
    server.lua = init_lua_env()
````

#### 20.5.2 SCRIPT EXISTS

SCRIPT EXSTS命令根据输入的SHA1校验和，检查校验和对应的脚本是否存在于服务器中。

SCRIPT EXSTS命令是通过检查给定的校验和是否存在于`lua_scripts`字典来实现的，以下是该命令的实现伪代码
````py
def SCRIPT_EXISTS(*sha1_list):

    # 结果列表
    result_list = []

    # 遍历输入的说有SHA1校验和
    for sha1 in sha1_list:

        if sha1 in server.lua_scripts:
            result_list.append(1)
        else:
            result_list.append(0)
    
    send_list_reply(result_list)
````

#### 20.5.3 SCRIPT LOAD

SCRIPT LOAD命令所做的事情和EVAL命令执行脚本时所做的前两步完全一样：
1. 命令首先在Lua环境中为脚本创建相对应的函数
2. 然后再将脚本保存到`lua_scripts`字典里面。
3. 最后将命令的sha1码返回给客户端

#### 20.5.4 SCRIPT KILL

如果服务器设置了`1ua-time-limit`配置选项，那么在每次执行Lua脚本之前，服务器都会在Lua环境里面设置一个超时处理钩子

超时处理钩子在脚本运行期间，会定期检查脚本已经运行了多长时间，一旦钩子发现脚本的运行时间已经超过了`1ua-time-1imit`选项设置的时长，钩子将定期在脚本运行的间隙中，查看是否有SCRIPT KILL命令或者SHUTDOWM命令到达服务器。

流程如下
![](Redis-20-Lua脚本\200308_7.png)

### 20.6 脚本复制

与其他普通Redis命令一样，当服务器运行在复制模式之下时，具有写性质的脚本命令也会被复制到从服务器

#### 20.6.1 复制EVAL命令、SCRIPT FLUSH命令和SCRIPT LOAD命令

Redis复制EVAL、SCRIPT FLUSH、SCRIPT LOAD三个命令的方法和复制其他普通Redis命令的方法一样，当主服务器执行完以上三个命令的其中一个时，主服务器会直接将被执行的命令传播给所有从服务器，如下图所示

![](Redis-20-Lua脚本\200308_8.png)

#### 20.6.2 复制EVALSHA命令

对于一个在主服务器被成功执行的EVALSHA命令来说，相同的EVALSHA命令在从服务器执行时却可能会出现脚本未找到错误。因为从服务器可能并没有载入这个脚本

因此Redis采取了如下的策略，来保证EVALSHA命令的安全

Redis要求主服务器在传播EVALSHA命令的时候，必须确保EVALSHA命令要执行的脚本已经被所有从服务器载入过，如果不能确保这一点的话，主服务器会将EVALSHA命令转换成一个等价的EVAL命令，然后通过传播EVAL命令来代替 EVALSHA命令。

##### 20.6.2.1 判断传播EVALSHA命令是否安全的方法

主服务器使用服务器状态的`repl_scriptcache_dict`字典记录自己已经将哪些脚本传播给了所有从服务器
````c
struct redisServer{
    //...

    dict *repl_scriptcache_dict;

    //...
}
````

`repl_scriptcache_dict`字典的键是一个个Lua脚本的SHA1校验和，而字典的值则全部都是`NULL`，当一个校验和出现在`repl_scriptcache_dict`字典时，说明这个校验和对应的Lua脚本已经传播给了所有从服务器，这种情况下，主服务器可以直接向从服务器传播包含这个SHA1校验和的EVALSHA命令

例如下图
![](Redis-20-Lua脚本\200308_9.png)

##### 20.6.2.2 清空repl_scriptcache_dict字典

每当主服务器添加一个新的从服务器时，主服务器都会清空自己的`repl_scriptcache_dict`字典，这是因为随着新从服务器的出现，`repl_scriptcache_dict`字典里面记录的脚本已经不再被所有从服务器载入过

##### 20.6.2.3 EVALSHA命令转换为EVAL命令的方法

通过使用EVALSHA命令指定的SHA1校验和，以及`lua_scripts`字典保存的Lua脚本，服务器总可以将一个EVALSHA命令
````sh
EVALSHA <sha1> <numkeys> [key..] [arg..] I
````

转换成一个等价的EVAL命令：
````sh
EVAL <script> <numkeys> [key..] [arg..]
````

##### 20.6.2.4 传播EVALSHA命令的方法

当主服务器在传播完EVAL命令之后，会将被传播脚本的sha1校验和添加到`repl_scriptcache_dict`字典里面，如果之后EVALSHA命令再次指定这个SHA1校验和，主服务器就可以直接传播EVALSHA命令，而不必再次对EVALSHA命令进行转换。

传播EVALSHA命令的流程图如下
![](Redis-20-Lua脚本\200308_10.png)
