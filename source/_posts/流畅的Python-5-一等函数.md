---
title: '[流畅的Python][5][一等函数]'
date: 2020-02-01 20:38:22
tags:
    - Python
categories:
    - 流畅的Python
---

## 第5章 一等函数

> 不管别人怎么说或怎么想，我从未觉得Python受到来自函数式语言的太多影响。我非常熟悉命令式语言，如C和Algol68，虽然我把函数定为一等对象，但是我并不把Python当成函数式编程语言      —————Guido van Rossum

- 在Python中，函数是一等对象

- 编程语言理论家把“一等函数”定义为满足下述条件的程序实体
    1. 运行时创建
    2. 能赋值给变量或数据结构中的元素
    3. 能作为参数传给函数
    4. 能作为函数的返回结果
---
### 5.1 把函数视为对象

- 示例 5-1：函数是一等对象
    ````py

    # 这是一个控制台会话，因此我们在运行时创建了一个函数
    >>> def factorial(n):
    ...     '''returns n'''
    ...     return 1 if n < 2 else n * factorial(n - 1)
    ... 
    >>> factorial(42)
    1405006117752879898543142606244511569936384000000000
    
    # __doc__是函数对象众多属性中的一个，用于生成对象的帮助文档
    >>> factorial.__doc__
    'returns n'
    
    # factorial是function类的一个实例
    >>> type(factorial)
    <class 'function'>
    ````

- `__doc__`属性生成对象的帮助文档。在Python交互式控制台中，可以调用help(obj)命令来查看这个文档

- 示例 5-2：将函数赋值给变量，再把函数作为参数传递
    ````py
    >>> fact = factorial
    >>> fact
    <function factorial at 0x7f84012f8e18>
    >>> fact(5)
    120
    >>> map(factorial, range(11))
    <map object at 0x7f83ff510860>
    >>> list(map(fact, range(11)))
    [1, 1, 2, 6, 24, 120, 720, 5040, 40320, 362880, 3628800]
    ````

- 以上两个示例可以充分说明在Python中函数是一等对象
---

### 5.2 高阶函数

- 高阶函数的定义：接受函数为参数，或者把函数作为结果返回的函数就是高阶函数(higher-order function)

- 例如：内置函数map和内置函数sorted

- 示例 5-3：高阶函数sorted的使用方法1
    ````py
    >>> fruits = ['strawberry', 'fig', 'apple', 'cherry', 'raspberry', 'banana']
    >>> sorted(fruits, key=len)
    ['fig', 'apple', 'cherry', 'banana', 'raspberry', 'strawberry']
    ````
    - 任何单参数函数都能作为key参数的值
---
#### 5.2.1 map、filter和reduce的现代替代品

- 函数式语言通常会提供map、filter和reduce三个高阶函数

- 在Python3中，map和filter还是内置函数，但是由于引入了列表推导和生成器表达式，它们变得不再那么重要

- 示例 5-5：map、filter与列表推导的对比
    ````py
    >>> list(map(fact, range(6)))
    [1, 1, 2, 6, 24, 120]
    >>> [fact(n) for n in range(6)]
    [1, 1, 2, 6, 24, 120]
    >>> list(map(fact, filter(lambda n: n % 2, range(6))))
    [1, 6, 120]
    >>> [fact(n) for n in range(6) if n % 2]
    [1, 6, 120]
    ````

- 在Python2中，reduce是内置函数，但是在Python3中放到了functools模块中

- 示例 5-6：使用reduce和sum计算0-99的和
    ````py
    >>> from functools import reduce
    >>> from operator import add
    >>> reduce(add, range(100))
    4950
    >>> sum(range(100))
    4950
    ````
    - sum和reduce的通用思想是把某个操作连续应用到序列的元素上，累计之前的结果，把一系列的值归约为一个值

- 归约函数
    - 把某个操作连续应用到序列的元素上，累计之前的结果，把一系列的值归约为一个值
    - 例如
        - `sum(iterable)`：对iterable中各个元素求和
        - `reduce()`
        - `all(iterable)`:如果iterable的每个元素都是真值，返回True,all([])返回True
        - `any(iterable)`：只要iterable中有元素时真值，就返回True,any([])返回False
---

### 5.3 匿名函数

- 匿名函数存在的原因：为了使用高阶函数，有时创建一次性的小型函数更便利

- 匿名函数简介
    - lambda关键字在Python表达式内创建匿名函数
    - 然而，Python简单的句法限制了lambda函数的定义体只能使用纯表达式。也就是，lambda函数的定义体中不能赋值，也不能使用while和try等Python语句
    
- 示例5-7：使用lambda表达式反转拼写，然后依次给单词列表排序
    ````py
    >>> fruits = ['strawberry', 'fig', 'apple', 'cherry', 'raspberry', 'banana']
    >>> sorted(fruits, key = lambda word: word[::-1])
    ['banana', 'apple', 'fig', 'raspberry', 'strawberry', 'cherry']
    ````

- 匿名函数的限制
    - 除了作为参数传给高阶函数之外，Python很少使用匿名函数
    - 由于句法上的限制，非平凡的lambda表达式要么难以阅读，要么无法写出

- Lundh提出的lambda表达式重构秘籍
    0. 如果使用lambda表达式导致一段代码难以理解,Fredrik Lundh建议像下面这样重构
    1. 编写注释，说明lambda表达式的作用
    2. 研究一会儿注释，病找出一个名称来概括注释
    3. 把lambda表达式转换成def，使用那个名称命名函数
    4. 删除注释
---

### 5.4 可调用对象
---

- 可调用对象简介
    - 可以运用调用运算符(即())调用的对象叫做可调用对象
    - 如果想判断一个对象是否可调用，可以使用内置的`callable()`函数
    - 可调用对象一共有7种

- 可调用对象分类

    | 可调用对象 | 简介 |
    | ------| ------ |
    | 用户定义的函数 | 使用`def`语句或`lambda`表达式创建 |
    | 内置函数 | 使用C语言(CPython)实现的函数，如`len`或`time.strftime` |
    | 内置方法 | 使用C语言实现的方法，如`dict.get` |
    | 方法 | 在类的定义体中定义的函数 |
    | 类 | 调用类时会产生类似构造方法的效果，返回一个此类的实体 |
    | 类的实例 | 如果类定义了`__call__`方法，那么它的实例可以作为函数调用 |
    | 生成器函数 | 使用`yield`关键字的函数或方法 |

---

### 5.5 用户定义的可调用类型

- 简介
    - 不仅Python函数是真正的对象，任何Python对象都可以表现得像函数
    - 为此，只需实现实例方法`__call__`

- 示例:实现了__call__方法的类，BingoCage
    - 源代码
        ````py
        import random

        class BingoCase:

            def __init__(self, item):
                # __init__接受任何可迭代对象;在本地构建一个副本,防止列表参数的意外副作用
                self._item = list(item)
                # shuffle定能完成工作,因为self._item是列表
                random.shuffle(self._item)

            # 起主要作用的方法
            def pick(self):
                try:
                    return self._item.pop()
                except IndexError:
                    # 如果self._item为空,抛出异常,并且设定错误信息
                    raise LookupError('pick from empty BingoCage')

            def __call__(self):
                # bingo.pick()的快捷方式是bingo()
                return self.pick()
        ````
    - 调用
        ````py
        >>> import bingocall
        >>> bingo = bingocall.BingoCase(range(3))
        >>> bingo.pick()
        2
        >>> bingo()
        0
        >>> callable(bingo)
        True
        ````

- 其他
    - 实现`__call__`方法的类是创建函数类对象的简便方法，此时必须在内部维护一个状态，让它在调用之间可用，例如BingoCage中的剩余元素
    - 创建保有内部状态的函数，还有一种截然不同的方式————使用闭包
---

### 5.6 函数内省

- 除了`__doc__`，函数对象还有很多属性，使用`dir`函数可以探知factorial具有下述属性
    ````py
    >>> def fun():
    ...     pass
    ... 
    >>> dir(fun)
    ['__annotations__', '__call__', '__class__', '__closure__', '__code__', '__defaults__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__get__', '__getattribute__', '__globals__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__kwdefaults__', '__le__', '__lt__', '__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
    >>> 
    ````

- `__dict__`属性
    - 函数使用`__dict__`属性储存赋予它的用户属性
    - 例如
        ````py
        >>> def fun():
        ...     pass
        ... 
        >>> fun.name = "haha"
        >>> fun.__dict__
        {'name': 'haha'}
        ````
    - 一般来说，为函数赋予属性不是很常见的做法

- 用户定义的函数的属性
    | 名称 | 类型 | 说明 |
    | ------| ------ | ------ |
    | `__annotations__` | `dict` | 参数和返回值的注释 |
    | `__call__` | `method-wrapper` | 实现`()`运算符；即可调用对象协议 |
    | `__closure__` | `tuple` | 函数闭包，即自由变量的绑定 |
    | `__code__` | `code` | 编译成字节码的函数元数据和函数定义体 |
    | `__defaults__` | `tuple` | 形式参数的默认值 |
    | `__get__` | `method-wrapper` | 实现制度描述符协议 |
    | `__globals__` | `dict` | 函数所在模块中的全局变量 |
    | `__kwdefaults__` | `dict` | 仅限关键字形式参数的默认值 |
    | `__name__` | `str` | 函数名称 |
    | `__qualname__` | `str` | 函数的限定名称，如`Random.choice` |

---

### 5.7 从定位参数到仅限关键字参数

- Python最好的特性之一是提供了及其灵活的参数处理机制

- 形式参数的分类
    - 定位参数：
        - 调用函数时根据函数定义的参数位置来传递参数
    - 关键字参数
        - 用于函数调用，通过“键-值”形式加以指定
        - 可以让函数更加清晰、容易使用，同时也清除了参数的顺序需求
        - 并且可以在函数声明时为其制定默认值
    - 可变参数：
        - 定义函数时，有时候我们不确定调用的时候会传递多少个参数(不传参也可以)
        - 此时，可用包裹(`*`或`**`)位置参数，或者包裹关键字参数，来进行参数传递，会显得非常方便

- `*`与`**`
    - `*`
        - 其他"剩余"参数会被放到一个元组中，被赋予带`*`的参数
        - 一个函数声明中只能有一个带`*`的参数
    - `**`
        - 带`**`的参数是一个字典，它会收集所有剩余关键字参数并放入一个字典中
        - 一个函数声明中只能有一个带`**`的参数

- 示例 5-10:tag函数定义部分,tag函数用来生成HTML标签
    ````py
    >>> import tag
    # 传入单个定位参数,生成一个指定名称的空标签
    >>> tag.tag('br')
    '<br />'
    # 第一个参数后面的任意个参数会被*content捕获,存入一个元组
    >>> tag.tag('p', 'hello')
    '<p>hello</p>'
    >>> tag.tag('p', 'hello', id=33)
    '<p id="33">hello</p>'
    # tag函数签名中没有明确指定名称的关键字参数会被**attrs捕获，存入一个字典
    >>> print(tag.tag('p', 'hello', 'world', cls='sidebar'))
    <p class="sidebar">hello</p>
    <p class="sidebar">world</p>
    # 调用tag函数时，即使第一个定位参数也能作为关键字参数传入
    >>> tag.tag(content='testing', name='img')
    '<img content="testing" />'
    >>> my_tag = {'name': 'img', 'title': 'Sunset Boulevard', 'src': 'sunset.jpg', 'cls': 'framed'}
    # 在my_tag前面加入**,字典中的所有元素作为单个参数传入，同名键会绑定到对应的具名参数上，余下的会被**attrs捕获
    >>> tag.tag(**my_tag)
    '<img class="framed" src="sunset.jpg" title="Sunset Boulevard" />'

    ````
---

### 5.8 获取关于参数的信息

- 举例：HTTP微框架Bobo中有个使用**函数内省**的好例子
    - 示例 5-12 `Bobo`知道`hello`需要`person`参数，并且从HTTP请求中获取它 
        ````py
        import bobo
        @bobo.query('/)
        def hello(person):
            return 'Hello %s!' % person
        ````
    - 解释
        - `bobo.query`装饰器把一个普通的函数(如`hello`)与框架的请求处理机制集成起来了
        - Bobo会内省`hello`函数，发现它需要一个名为`person`的参数，然后从请求中获取那个名称对应的参数，将其传给`hello`函数，因此程序员**根本不需要**触碰请求对象

- 那么问题来了
    - Bobo是怎么知道函数需要哪个参数的呢？它又是怎么知道参数有没有默认值呢
    - 解答
        - 函数对象有个`__default__`属性，它的值是一个元组，里面保存了定位参数和关键字参数的默认值
        - 仅限关键字参数的默认值在`__kwdefault__`属性中
        - 参数的名称在`__code__`属性中，它的值是一个code对象引用，自身也有很多属性
        - 但是这种组织信息的方式并不便利，我们可以使用`inspect`模块来更好的提取参数信息
---

### 5.9 函数注解

- 函数注解的用法
    - 函数声明中的各个参数可以在`:`之后增加注解表达式
    - 如果参数有默认值，注解放在参数名和`=`之间
    - 如果想注解返回值，在`)`和函数声明末尾的`:`之间添加`->`和一个表达式。那个表达式可以是任意类型。注解中常用的类型是类(如str或int)和字符串(如'int > 0')

- 示例
    - 不加注解的函数声明版本
        ````py
        def clip(text, max_len):
        ````
    - 加注解的函数声明版本
        ````py
        def clip(text:str, max_len:'int > 0'=80) -> str:
        ````

- Python对注解所做的处理
    - Python对注解所做的唯一的事情就是把它们存储在函数的__annotations__属性里，仅此而已
    - Python不做检查，不做强制，不做验证，什么操作都不做
    - 注解在Python解释器中没有任何意义，注解只是元数据，可以供IDE、框架和装饰器等工具使用

---

### 5.10 本章小结

1. 本章的目标是探讨Python函数的一等本性。这意味着，我们可以把函数赋值给变量、传给其他函数、存储在数据结构中，以及访问函数的属性，供框架和一些工具使用

2. 高阶函数

3. Python有7种可调用对象，这些可调用对象都能通过内置的`callable()`函数检测

4. 每种可调用对象都支持使用相同的丰富句法声明形式参数，包括仅限关键字参数和注解

5. Python函数及其注解有丰富的属性，在inspect模块的帮助下，可以读取它们

7. 函数式编程可以使用`operator`模块中的一些函数，以及`functools.partial`函数
---

