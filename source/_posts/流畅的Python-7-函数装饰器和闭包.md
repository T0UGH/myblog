---
title: '[流畅的Python][7][函数装饰器和闭包]'
date: 2020-02-01 20:42:08
tags:
    - Python
categories:
    - 流畅的Python
---

## 第七章 函数装饰器和闭包

- 函数装饰器用于在源码中"标记"函数，以某种方式增强函数的行为

- `nonlocal`是新近出现的保留关键字，如果想实现函数装饰器必须了解`nonlocal`

- 除了在装饰器中有用处之外，闭包还是回调式异步编程和函数式编程风格的基础
---
### 7.1 装饰器基础知识

- 简介
    - 装饰器是可调用的对象，其参数是另一个函数(被装饰的函数)。
    - 装饰器可能会处理被装饰的函数，然后把它返回，或者将其替换成另一个函数或可调用对象

- 严格来说，装饰器就是一种语法糖
    - 下面两种写法的效果是一样的
        ````py
        @decorate
        def target():
            print('running target()')
        ````
        ````py
        def target():
            print('running target()')
        target = decorate(target)
        ````
    - 如上所示装饰器可以像常规的可调用对象那样调用，其参数是另一个函数

- 示例：装饰器通常把函数替换为另一个函数
    ````py
    >>> def deco(func):
    ...     def inner():
    ...             print('running inner()')
            # deco返回inner函数对象
    ...     return inner
    ... 
    # 使用deco装饰target
    >>> @deco
    ... def target():
    ...     print('running target()')
    ... 
    # 调用被装饰的target实际会运行inner
    >>> target()
    running inner()
    # 审查对象，发现target现在是inner的引用
    >>> target
    <function deco.<locals>.inner at 0x7f8fc02efb70>
    ````

- 装饰器的两个特性
    1. 能把被装饰的函数替换为其他函数
    2. 装饰器在加载模块时立即执行

---

### 7.2 Python何时执行装饰器

- 装饰器的一个关键特性是，它们在被装饰的函数定义之后立即运行。这通常是在导入时

- 示例：registration.py模块
    ````py
    # 使用registry保存被@register装饰的函数引用
    registry = []


    # register的参数是一个函数
    def register(func):
        print('running register (%s)' % func)
        # 把func传入registry
        registry.append(func)
        # 返回func:必须返回函数,这里返回的函数与通过参数传入的一样
        return func


    # f1和f2被register装饰
    @register
    def f1():
        print("running f1()")


    @register
    def f2():
        print("running f2()")


    # f3没有被register装饰
    def f3():
        print("running f3()")


    # main显示registry,然后调用f1(),f2()和f3()
    def main():
        print('running main()')
        print('registry ->', registry)
        f1()
        f2()
        f3()


    if __name__ == '__main__':
        main()

    '''
    running register (<function f1 at 0x7f43d476a510>)
    running register (<function f2 at 0x7f43d476a7b8>)
    running main()
    registry -> [<function f1 at 0x7f43d476a510>, <function f2 at 0x7f43d476a7b8>]
    running f1()
    running f2()
    running f3()
    '''
    ````
- 注意
    - 函数装饰器在导入模块时立即执行
    - 而被装饰的函数只在明确调用时执行
---

### 7.3 使用装饰器改进策略模式

- 使用注册装饰器可以改进6.1节中的电商促销折扣示例

- 示例:使用装饰器来实现策略模式
    ````py
    # promos列表最初是空的
    promos = []


    # promotion把promo_func添加到promos列表中,然后原封不动地将其返回
    def promotion(promo_func):
        promos.append(promo_func)
        return promo_func


    # 被@promotion装饰的函数都会添加到promos列表中
    @promotion
    def fidelity(order):
        """为积分为1000或以上的顾客提供5%折扣"""
        return order.total() * .05 if order.customer.fidelity >= 1000 else 0


    @promotion
    def bulk_item(order):
        """单个商品为20个或以上时提供10%折扣"""
        discount = 0
        for item in order.cart:
            if item.quantity >= 20:
                discount += item.total() * .1
        return discount


    @promotion
    def large_order(order):
        """订单中的不同商品达到10个或以上时提供7%折扣"""
        distinct_items = {item.product for item in order.cart}
        if len(distinct_items) >= 10:
            return order.total() * .07
        return 0


    def best_promo(order):
        """选择可用的最佳折扣"""
        return max(promo(order) for promo in promos)
    ````

- 这个方案的优点
    1. 促销策略函数无需使用特殊的名称(即不用以_promo结尾)
    2. `@promotion`装饰器突出了被装饰函数的作用，还便于临时禁用某个促销策略：只需要把装饰器注释掉
    3. 促销折扣策略可以在其他模式中定义，在系统的任何地方都行，只要使用`@promotion`装饰即可
---

### 7.4 变量作用域规则

- 变量的类型
    - 全局变量：直接在模块中定义的变量，即在函数或方法等可调用对象外部定义的变量
    - 局部变量：也称全局变量，在函数或方法等可调用对象外部定义的变量

- 第一个例子：读取一个局部变量和一个全局变量
    ````py
    # b是一个全局变量
    >>> b = 6
    >>> def f1(a):
            # a是函数f1内部的变量,是局部变量
    ...     print(a)
    ...     print(b)
    ... 
    >>> f1(3)
    3
    6
    ````

- 第二个例子：一个让人吃惊的例子
    ````py
    >>> b = 6
    >>> def f2(a):
    ...     print(a)
    ...     print(b)
    ...     b = 9
    ... 
    >>> f2(3)
    3
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "<stdin>", line 3, in f2
    UnboundLocalError: local variable 'b' referenced before assignment
    ````
    - 解释
        - 在这个例子中,b被当做了局部变量
        - 因为在函数中给它赋值了
        - 所以报错，`引用在赋值之前`
    - 这不是缺陷，而是设计选择
        - Python不要求声明变量，但是假定在函数定义体中赋值的变量是局部变量
        - 这比JavaScript的行为要好得多，JavaScript也不要求声明变量，但是如果忘记把变量声明为局部变量(使用var),可能会在不知情的情况下获取全局变量
    - 如果想在函数赋值中把b当做全局变量，要使用global声明`global b`
---

### 7.5 闭包

- 闭包定义
    - 闭包指延伸了作用域的函数，其中包含函数定义体中引用、但是不在定义体中定义的非全局变量
    - 函数是不是匿名的没有关系，关键是它能访问定义体之外定义的的非全局变量

- 举例
    - 背景：假如有个名为avg的函数，它的作用是计算不断增加的系列值的均值
    - 使用方式
        ````py
        >>> avg(10)
        10.0
        >>> avg(11)
        10.5
        >>> avg(12)
        11.0
        ````
    - 函数式实现
        ````py
        def make_averager():
            series = []

            def averager(new_value):
                series.append(new_value)
                total = sum(series)
                return total / len(series)

            return averager
        ````
    - 使用类实现
        ````py
        class Averager():
            def __init__(self):
                self.series = []
            
            def __call__(self, new_value):
                self.series.append(new_value)
                total = sum(self.series)
                return total / len(self.serious)
        ````
    - 两个示例的共通之处
        - 调用Averager()或make_averager()得到一个可调用对象avg,它会更新历史值，然后计算当前均值
        - 我们都需要把n放到系列值中，然后重新计算均值
    
    - 问题：那么函数式实现时是怎么找到`series`的呢
        - 在averager函数中，series是自由变量
        - 自由变量：指未在本地作用域中绑定的变量
        - 也就是说，虽然定义这个变量的函数已经返回了，但是这个变量被绑定到了内部函数上

- 综上
    - 闭包是一种函数，它会保留定义函数时存在的自由变量的绑定
    - 这样调用函数时，虽然定义作用域不可用了，但是仍然使用那些绑定
    - 只有嵌套在其他函数中的函数才可能需要处理不在全局作用域中的外部变量
---

### 7.6 nonlocal声明

- 背景
    - 前面实现make_averager函数的方法效率不高
    - 更好的实现方式是，只存储目前的总值和元素个数，然后使用这两个数计算均值

- 一种有缺陷的实现
    ````py
    >>> def make_averager():
    ...     count = 0
    ...     total = 0
    ...     def averager(new_value):
    ...             count += 1
    ...             total += new_value
    ...             return total / count
    ...     return averager
    ... 
    >>> avg = make_averager()
    >>> avg(10)
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "<stdin>", line 5, in averager
    UnboundLocalError: local variable 'count' referenced before assignment
    ````

- 问题
    - `count += 1`为count赋值，所以count被当做局部变量而不是自由变量，`count += 1`相当于`count = count + 1`,`count + 1`先于`count`执行，所以报错了`local variable 'count' referenced before assignment`

- 解决方法
    - python3引入了nonlocal关键字，如同global关键字，它的作用是把变量标记为自由变量
    - 即使在函数中为变量赋予新值了，也会变成自由变量

- 对上面有缺陷实现的改进
    ````py
    ````py
    >>> def make_averager():
    ...     count = 0
    ...     total = 0
    ...     def averager(new_value):
    ...             nonlocal count, total
    ...             count += 1
    ...             total += new_value
    ...             return total / count
    ...     return averager
    ... 
    ````
---

### 7.7 实现一个简单的装饰器

- 示例1：定义了一个装饰器,它会在每次调用被装饰的函数时计时,然后把经过的时间、传入的参数和调用的结果打印出来。
    ````py
    import time

    def clock(func):
        # 定义内部函数clocked,它接受任何个定位参数
        def clocked(*args):
            t0 = time.perf_counter()
            # 下面的代码可以执行,因为clocked闭包包含自由变量func
            result = func(*args)
            elapsed = time.perf_counter() - t0
            name = func.__name__
            arg_str = ', '.join(repr(arg) for arg in args)
            print('[%0.8fs]%s(%s) -> %r' % (elapsed, name, arg_str, result))
            return result
        # 返回内部函数，取代被装饰的函数
        return clocked
    ````
- 示例2：对示例1的使用
    ````py
    import time
    from clockdeco import clock


    @clock
    def snooze(seconds):
        time.sleep(seconds)


    @clock
    def factorial(n):
        return 1 if n < 2 else n * factorial(n - 1)


    if __name__ == '__main__':
        print('*' * 40, 'Calling snooze(.123)')
        snooze(.123)
        print('*' * 40, 'Calling snooze(6)')
        print('6! =', factorial(6))
    
    '''
    **************************************** Calling snooze(.123)
    [0.12314407s]snooze(0.123) -> None
    **************************************** Calling snooze(6)
    [0.00000057s]factorial(1) -> 1
    [0.00001018s]factorial(2) -> 2
    [0.00001566s]factorial(3) -> 6
    [0.00002063s]factorial(4) -> 24
    [0.00002564s]factorial(5) -> 120
    [0.00003240s]factorial(6) -> 720
    6! = 720
    '''
    ````

- 工作原理
    - factorial会作为func参数传给clock.然后clock函数返回clocked函数，python解释器在背后将clocked赋值给factorial
    - 自此,每次调用factorial(n)执行的都是clocked(n)
    - clocked(n)大致做了下面几件事
        1. 记录初始时间 t0。
        2. 调用原来的 factorial 函数,保存结果。
        3. 计算经过的时间。
        4. 格式化收集的数据,然后打印出来。
        5. 返回第 2 步保存的结果。

- 装饰器的典型行为
    - 如上例，装饰器往往会把被装饰的函数替换成新函数,二者接受相同的参数,而且(通常)返回被装饰的函数本该返回的值
    - 但是通常还会做些额外操作

- 示例1的缺点
    1. 不支持关键字参数
    2. 遮盖了被装饰函数的 `__name__` 和 `__doc__` 属性

- 示例3：使用`functools.wraps`装饰器把相关的属性从`func`复制到`clocked`中，并且可以处理关键字参数
    ````py
    # clockdeco2.py
    import time
    import functools


    def clock(func):
        @functools.wraps(func)
        def clocked(*args, **kwargs):
            t0 = time.time()
            result = func(*args, **kwargs)
            elapsed = time.time() - t0
            name = func.__name__
            arg_lst = []
            if args:
                arg_lst.append(', '.join(repr(arg) for arg in args))
            if kwargs:
                pairs = ['%s=%r' % (k, w) for k, w in sorted(kwargs.items())]
            arg_lst.append(', '.join(pairs))
            arg_str = ', '.join(arg_lst)
            print('[%0.8fs] %s(%s) -> %r ' % (elapsed, name, arg_str, result))
            return result
    return clocked
    ````
---

### 7.8 标准库中的装饰器

- Python内置了三个用于装饰方法的装饰器`property`、`classmethod` 和 `staticmethod`

- `functools`模块中几个好用的装饰器
    - `functools.wraps`
    - `functools.lru_cache`
    - `functools.singledispatch`

---
#### 7.8.1 使用functools.lru_cache做备忘

- 简介
    - `functools.lru_cache` 是非常实用的装饰器,它实现了备忘(memoization)功能。
    - 这是一项优化技术,它把耗时的函数的结果保存起来,避免传入相同的参数时重复计算。
    - LRU 三个字母是“Least Recently Used”的缩写,表明缓存不会无限制增长,一段时间不用的缓存条目会被扔掉。
    - 例如，`fibonacci`中著名的指数爆炸问题，可以通过`functools.lru_cache`建立缓存来解决

- 示例:使用缓存来优化`fibonacci`算法
    ````py
    import functools

    from clockdeco import clock

    # 注意，这里必须加上括号来调用，原因参见7.10
    @functools.lru_cache()
    # 这里叠放了装饰器：@lru_cache()应用到@clock返回的函数，具体参见7.9
    @clock
    def fibonacci(n):
        if n < 2:
            return n
        return fibonacci(n - 2) + fibonacci(n - 1)


    if __name__ == '__main__':
        print(fibonacci(6))

    # 这里的输出显示，因为递归导致的大量重复计算的问题得到了解决
    '''
    /usr/bin/python3.6 /home/tough/code/pycharm/helloworld/fibonacci_cache_version.py
    [0.00000046s]fibonacci(0) -> 0
    [0.00000050s]fibonacci(1) -> 1
    [0.00003141s]fibonacci(2) -> 1
    [0.00000100s]fibonacci(3) -> 2
    [0.00004295s]fibonacci(4) -> 3
    [0.00000067s]fibonacci(5) -> 5
    [0.00005414s]fibonacci(6) -> 8
    8
    '''
    ````

- lru_cache的两个可选参数
    - 完整的函数签名:`functools.lru_cache(maxsize=128, typed=False)`
    - `maxsize`
        - `maxsize`参数指定存储多少个调用的结果
        - 缓存满了之后,旧的结果会被扔掉,腾出空间
        - 为了得到最佳性能,`maxsize`应该设为2的幂
    - `typed`
        - `typed` 参数如果设为 `True`,把不同参数类型得到的结果分开保存,即把通常认为相等的浮点数和整数参数(如 1 和 1.0)区分开。
    - 顺便说一下:因为 `lru_cache` 使用字典存储结果,而且键根据调用时传入的定位参数和关键字参数创建,所以被 `lru_cache` 装饰的函数,它的所有参数都必须是**可散列的**。

---
#### 7.8.2 单分派泛函数

- 背景
    - 假设我们在开发一个调试 Web 应用的工具,我们想生成 HTML,显示不同类型的 Python 对象
    - 这个函数适用于任何 Python 类型,但是它需要使用特别的方式显示某些类型
        - str:把内部的换行符替换为 `'<br>\n'`;不使用 `<pre>`,而是使用 `<p>`
        - int:以十进制和十六进制显示数字
        - list:输出一个 HTML 列表,根据各个元素的类型进行格式化
    - 我们想要的行为如下所示
        ````py
        # 默认格式
        >>> htmlize({1, 2, 3}) 
        '<pre>{1, 2, 3}</pre>'
        >>> htmlize(abs)
        '<pre><built-in function abs></pre>'
        # 字符串格式
        >>> htmlize('Heimlich & Co.\n- a game') 
        '<p>Heimlich & Co.<br>\n- a game</p>'
        # 数字格式
        >>> htmlize(42) 
        '<pre>42 (0x2a)</pre>'
        # 列表格式
        >>> print(htmlize(['alpha', 66, {3, 2, 1}]))
        <ul>
        <li><p>alpha</p></li>
        <li><pre>66 (0x42)</pre></li>
        <li><pre>{1, 2, 3}</pre></li>
        </ul>
        ````

- 分析
    1. 因为 Python 不支持重载方法或函数,所以我们不能使用不同的签名定义htmlize 的变体,也无法使用不同的方式处理不同的数据类型。
    2. 在 Python 中,一种常见的做法是把 htmlize 变成一个分派函数
        - 使用一串 if/elif/elif,调用专门的函数,如 htmlize_str、htmlize_int,等等
        - 这样不便于模块的用户扩展,还显得笨拙:时间一长,分派函数 htmlize 会变得很大,而且它与各个专门函数之间的耦合也很紧密。
    3. Python 3.4 新增的 functools.singledispatch 装饰器可以把整体方案拆分成多个模块
        - 使用`@singledispatch`装饰的普通函数会变成泛函数(generic function)
        - 泛函数：根据第一个参数的类型,以不同方式执行相同操作的一组函数。

- 解决：示例：`singledispatch`创建一个自定义的`htmlize.register`装饰器,把多个函数绑在一起组成一个泛函数
    ````py
    from functools import singledispatch
    from collections import abc
    import numbers
    import html


    # @singledispatch标注处理object类型的基函数
    @singledispatch
    def htmlize(obj):
        content = html.escape(repr(obj))
        return '<pre>{}</pre>'.format(content)


    # 各个专门函数使用 @«base_function».register(«type») 装饰
    # 专门函数的名称无关紧要; _是个不错的选择,简单明了
    @htmlize.register(str)
    def _(text):
        content = html.escape(text).replace('\n', '<br>\n')
        return '<p>{0}</p>'.format(content)


    # 为每个需要特殊处理的类型注册一个函数。numbers.Integral 是 int 的虚拟超类
    @htmlize.register(numbers.Integral)
    def _(n):
        return '<pre>{0} (0x{0:x})</pre>'.format(n)


    # 可以叠放多个 register 装饰器,让同一个函数支持不同类型
    @htmlize.register(tuple)
    @htmlize.register(abc.MutableSequence)
    def _(seq):
        inner = '</li>\n<li>'.join(htmlize(item) for item in seq)
        return '<ul>\n<li>' + inner + '</li>\n</ul>'
    ````

- 优点
    - `singledispatch`机制的一个显著特征是,你可以在系统的任何地方和任何模块中注册专门函数。如果后来在新的模块中定义了新的类型,可以轻松地添加一个新的专门函数来处理那个类型。
    - 此外,你还可以为不是自己编写的或者不能修改的类添加自定义函数。
    - `singledispath`支持模块化扩展:各个模块可以为它支持的各个类型注册一个专门函数。

---

### 7.9 叠放装饰器

- 简介
    - 即可以在已经被装饰的函数上应用装饰器
    - 把`@d1`和`@d2`两个装饰器按顺序应用到`f`函数上,作用相当于`f=d1(d2(f))`

---

### 7.10 参数化装饰器

- 背景
    - 解析源码中的装饰器时,Python 把被装饰的函数作为第一个参数传给装饰器函数。
    - 那怎么让装饰器接受其他参数呢?
        - 答案是，创建一个装饰器工厂函数,
        - 把参数传给它,返回一个装饰器,然后再把它应用到要装饰的函数上

---
#### 7.10.1 一个参数化的注册装饰器
- 我们首先看一个最简单的装饰器
    ````py
    registry = []
    
    
    def register(func):
        print('running register(%s)' % func)
        registry.append(func)
        return func

    
    @register
    def f1():
        print('running f1()')
        print('running main()')
        print('registry ->', registry)
    
    f1()
    ````
- 假如需要为register提供一个可选的active参数,当设为false时，不注册被装饰的函数
    ````py
    
    registry = set() 
    
    # register 接受一个可选的关键字参数
    def register(active=True): 
        # decorate 这个内部函数是真正的装饰器
        def decorate(func): 
            print('running register(active=%s)->decorate(%s)' % (active, func))
            # 只有 active 参数的值(从闭包中获取)是 True 时才注册 func
            if active:
            registry.add(func)
            # 如果 active 不为真,而且 func 在 registry 中,那么把它删除
            else:
            registry.discard(func)
            # decorate 是装饰器,必须返回一个函数
            return func
        # register 是装饰器工厂函数,因此返回 decorate
        return decorate


    # @register 工厂函数必须作为函数调用,并且传入所需的参数
    @register(active=False) 
    def f1():
    print('running f1()')
    

    # 即使不传入参数,register 也必须作为函数调用
    @register() 
    def f2():
    print('running f2()')
    
    ````
    - 所以为什么调用时一定要加括号，是因为`register`此时并不是装饰器函数而是装饰器的工厂函数，只有加`()`才可以返回真正的装饰器函数
---
#### 7.10.2 参数化clock装饰器

- 本节再次讨论clock装饰器,为它添加一个功能：让用户传入一个格式字符串，控制被装饰函数的输出

- 示例：参数化clock装饰器
    ````py
    import time
    
    DEFAULT_FMT = '[{elapsed:0.8f}s] {name}({args}) -> {result}'
    
    
    def clock(fmt=DEFAULT_FMT):
        def decorate(func):
            def clocked(*_args): 
                t0 = time.time()
                _result = func(*_args) 
                elapsed = time.time() - t0
                name = func.__name__
                args = ', '.join(repr(arg) for arg in _args)
                result = repr(_result) 
                print(fmt.format(**locals()))
                return _result 
            return clocked 
        return decorate 

    if __name__ == '__main__':
        @clock()
        def snooze(seconds):
        time.sleep(seconds)
        for i in range(3):
            snooze(.123)
    ````

- 装饰器还可以通过实现__call_方法的类实现
    ````py

    class register():

        def __init__(self, active=True):
            self.active = active

        def __call__(self, func):
            print('running register(active=%s)->decorate(%s)' % (self.active, func))
            return func


    @register(active=False)
    def f1():
        print('running f1()')


    # 如果这里可以给一个类的实例或许不需要加括号
    # 即使不传入参数,register 也必须作为函数调用
    @register()
    def f2():
        print('running f2()')


    if __name__ == '__main__':
        print(type(f1))
        f1()
        print(type(f2))
        f2()


    '''
    running register(active=False)->decorate(<function f1 at 0x7f269489be18>)
    running register(active=True)->decorate(<function f2 at 0x7f2692aab840>)
    running f1()
    running f2()
    '''
    ````
---


