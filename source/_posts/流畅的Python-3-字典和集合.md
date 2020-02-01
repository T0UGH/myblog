---
title: '[流畅的Python][3][字典和集合]'
date: 2020-02-01 20:34:21
tags:
    - Python
categories:
    - 流畅的Python
---
## 第3章 字典和集合

> 字典这个数据结构活跃在所有的python程序背后，即便你的源码中并没有直接用到它。---A.M.Kuching

- dict类型不但在各种程序里广泛使用，它也是Python语言的基石

- 模块的命名空间、实例的属性和函数的关键字参数里都可以看到字典的影子

- 正因为字典至关重要，Python对它的实现做了高度优化，而散列表则是字典类型性能出众的根本原因

- 集合(set)的实现也依赖于散列表
---
### 3.1 泛映射类型

- 抽象基类
    - collections.abc模块中有Maping和MutableMapping这两个抽象基类，它们的作用是为dict和其他类似的类型定义**形式接口**，如图
    ![]

    - 然而，非抽象映射类型一般不会直接继承这些抽象基类，它们会直接对dict或是collections.UserDict进行扩展。这些抽象基类的主要作用是作为形式化文档，它们定义了构建一个映射类型所需要的最基本的接口。

    - 他们还可以跟isinstance一起被用来判定某个数据是不是广义上的映射类型
        ````py
        my_dict = {}
        print(isinstance(my_dict,abc.Mapping))
        # True
        ````
- 标准库里的所有映射类型都是利用**dict**来实现的，因此它们有个共同的限制，即只有**可散列的数据类型**才能作为这些映射里的键

- 什么是可散列的数据类型
    - 如果一个对象是可散列的，那么在这个对象的生命周期中，它的散列值是不变的，而且这个对象需要实现`__hash()__`方法
    - 另外可散列对象还要有`__eq()__`方法，这样才能跟其他键做比较
    - **原子不可变数据类型**都是可散列的,如：`str`、`bytes`和数值类型
    - **元组**的话，只有当一个元组包含的所有元素都是可散列类型的情况下，它才是可散列的
    - 一般来讲**用户自定义的数据类型**都是可散列的，散列值就是它们的`id()`函数的返回值，所以所有这些对象在比较的时候都是不相等的

- 字典的多种**构造方法**
````py
a = dict(one=1, two=2, three=3)
b = {'one': 1, 'two': 2, 'three': 3}
c = dict(zip(['one' ,'two' ,'three'], [1, 2, 3]))
d = dict([('two', 2), ('one', 1), ('three',3)])
e = dict({'one': 1, 'two': 2, 'three': 3})
print(a==b==c==d==e)
# True
````

### 3.2 字典推导

- 字典推导可以从任何以键值对作为元素的可迭代对象中构建出字典

- dictcompDemo
    ````py
    DIAL_CODES = [
        (86, 'China'),
        (91, 'India'),
        (1, 'United States'),
        (62, 'Indonesia'),
        (55, 'Brazil'),
        (92, 'Pakistan'),
        (880, 'Bangladesh'),
        (234, 'Nigeria'),
        (7, 'Russia'),
        (81, 'Japan')
    ]

    # 这里使用国家名作为key,国家码作为value
    country_code = {country: code for code, country in DIAL_CODES}
    print(country_code)
    # 这里用国家码作为key,并且将国家名称转化为大写,过滤掉区域码大于或等于66的地区
    code_country = {code: country.upper() for country, code in country_code.items() if code < 66}
    print(code_country)
    '''
    {'China': 86, 'India': 91, 'United States': 1, 'Indonesia': 62, 'Brazil': 55, 'Pakistan': 92, 'Bangladesh': 880, 'Nigeria': 234, 'Russia': 7, 'Japan': 81}
    {1: 'UNITED STATES', 62: 'INDONESIA', 55: 'BRAZIL', 7: 'RUSSIA'}
    '''
    ````

### 3.3 常见的映射方法
---

- dict、defaultdict和OrderedDict的常见方法
    - 如图
    ![]()
    - update方法处理参数m的方式，是典型的“鸭子类型”
        1. 函数首先检查m是否有keys方法，如果有，那么update函数就把它当作映射对象来处理
        2. 否则，函数会退一步，转而把m当作包含了键值对(key, value)元素的迭代器
        3. 因此你既可以用一个映射对象来新建一个映射对象，也可以用包含(key, value)元素的可迭代对象来初始化一个映射对象

- 用setdefault处理找不到的键
    - d.setdefault(k, [default])：若字典中有键k，则直接返回k所对应的值；若无，则让d[k]=default，然后返回default
    - setdefaultDemo
        ````py
        import sys
        import re

        # 这段程序用来从文件中统计每个单词出现的位置

        # 声明一个选择单词的正则式
        WORD_RE = re.compile(r'\w+')

        # 初始化index字典
        index = {}

        # 打开文件并按行迭代每个单词
        with open(sys.argv[1], encoding='utf-8') as fp:
            for line_no, line in enumerate(fp, 1):
                for match in WORD_RE.finditer(line):
                    word = match.group()
                    # 获得这个单词的列坐标
                    column_no = match.start() + 1
                    location = (line_no, column_no)
                    # 获得单词的出现情况表，如果单词不存在，把单词和一个空列表放进映射，然后返回这个空列表，这样就能在不进行第二次查找的情况下更新列表了
                    index.setdefault(word, []).append(location)

        # 查看结果
        for word in sorted(index, key=str.upper):
            print(word, index[word])
        ````
- 
---

### 3.4 映射的弹性键查询

- 就算某个键在映射中不存在，我们还是希望在通过这个键读取值的时候能得到一个默认值

- 有两种途径可以帮我们达到这个目的
    1. 通过defaultdict这个类型而不是普通的dict
    2. 给自己定义一个dict的子类，然后在子类中实现`__missing__`方法
---
#### 3.4.1 defaultdict:处理找不到的键的一个选择

- 简介：
    - 在实例化一个defaultdict的时候，需要给构造方法提供一个可调用对象
    - 这个可调用对象会在__getitem__碰到找不到的键的时候被调用，让`__getitem__`返回某种默认值

- 例如:我们新建一个这样的字典,`dd = defaultdict(list)`，如果键'new-key'在dd中不存在的话，表达式`dd['new-key']`会按照以下步骤来行事
    1. 调用list()来建立一个新列表
    2. 把这个新列表作为值，'new-key'作为它的键，放在dd中
    3. 返回这个列表的引用

- 如果在创建defaultdict的时候没有指定default_factory，查询不存在的键会触发keyError

- defaultdict里的default_factory只会在`__getitem__`里被调用，在其他方法里完全不会发挥作用
    - 比如，dd是个defaultdict,k是个找不到的键，dd[k]这个表达式会调用default_factory来创建一个默认值，而dd.get(k)则会返回None

- defaultdictDemo
    ````py
    import sys
    import re
    import collections

    # 这段程序用来从文件中统计每个单词出现的位置

    # 声明一个选择单词的正则式
    WORD_RE = re.compile(r'\w+')

    # 初始化index字典
    index = collections.defaultdict(list)

    # 打开文件并按行迭代每个单词
    with open(sys.argv[1], encoding='utf-8') as fp:
        for line_no, line in enumerate(fp, 1):
            for match in WORD_RE.finditer(line):
                word = match.group()
                # 获得这个单词的列坐标
                column_no = match.start() + 1
                location = (line_no, column_no)
                # 获得此单词在字典中的value,若之前不存在就设置为[],并返回
                index[word].append(location)

    # 查看结果
    for word in sorted(index, key=str.upper):
        print(word, index[word])
    ````
---
#### 3.4.2 特殊方法`__missing__`

- 所有的映射类型在处理找不到的键的时候，都会牵扯到`__missing__`方法

- 虽然基类dict并没有定义这个方法，但是dict是知道有这么个东西存在的
    - 也就是说，如果有一个类继承了dict，然后这个继承类提供了`__missing__`方法，那么在`__getitem__`碰到找不到的键的时候，Python会自动调用它，而不是抛出一个KeyError异常

- `__missing__`方法只会被`__getitem__`调用，提供__missing__方法对`get`或者`__contains__`这些方法的使用没有影响

- missingDemo
    ````py
    # StrKeyDict在查询的时候会把非字符串的键转换为字符串


    # StrKeyDict继承自dict
    class StrKeyDict0(dict):

        def __missing__(self, key):
            # 如果找不到的键本身就是字符串,那就抛出KeyError异常
            if isinstance(key, str):
                raise KeyError(key)
            # 如果找不到的键不是字符串,就把它转换为字符串再查找
            return self[str(key)]

        def get(self, key, default=None):
            try:
                # get方法把查找工作用self[key]的形式委托给`__getitem__`，这样就宣布了查找失败之前,还能通过`__missing__`再给某个键一个机会
                return self[key]
            except KeyError:
                # 如果抛出异常,那么说明__missing__也失败了,于是返回default
                return default

        def __contains__(self, item):
            # 先按照传入键的原本的值来查找，如果没有找到，再用str()方法把键转换成字符串再查找一次
            return item in self.keys() or str(item) in self.keys()


    # 调用尝试1
    d = StrKeyDict0([('2', 'two'), (4, 'four')])
    print(d['2'])
    print(d[4])
    print(d[1])
    '''
    two
    four
    Traceback (most recent call last):
    ...
    KeyError: '1'
    '''

    # 调用尝试2
    print(d.get('2'))
    print(d.get(4))
    print(d.get(1, 'N/A'))
    '''
    two
    four
    N/A
    '''

    # 调用尝试3
    print(2 in d)
    print(1 in d)
    '''
    True
    False
    '''
    ````

- 像 k in my_dict().keys()这种操作在Python3中是很快的，因为dict.keys()的返回值是一个'视图'，'视图'就像一个集合，而且跟字典类似的是，在视图里查找一个元素的速度很快
---

### 3.5 字典的变种

1. collections.OrderedDict
    - 这个类型在添加键的时候会保持顺序，因此键的迭代次序总是一致的

2. collectinos.ChainMap
    - 该类型可以容纳数个不同的映射对象，然后在进行键查找操作的时候，这些对象会被当作一个整体被逐个查找，直到键被找到为止
    - 例如
    ````py
    import builtins
    pylookup = ChainMap(local(), global(), vars(builtins))
    ````

3. collections.Counter
    - 这个映射类型会给键准备一个整数计数器
    - 每次更新一个键的时候都会增加这个计数器
    - 所以这个类型可以用来给可散列对象计数，或者被当成多重集来用
    - demo
    ````python
    >>> import collections
    >>> ct = collections.Counter('abracadabra')
    >>> ct
    Counter({'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
    >>> ct.update('aaaaazzz')
    >>> ct
    Counter({'a': 10, 'z': 3, 'b': 2, 'r': 2, 'c': 1, 'd': 1})
    # most_common会按照次序返回映射中最常用的n个键和它们的计数
    >>> ct.most_common(2)
    [('a', 10), ('z', 3)]

    ````

4. collections.UserDict
    - 这个类其实就是把标准dict用纯Python的方式又实现了一编
---

### 3.6 子类化UserDict
---
- 就创造自定义映射类型来说，以UserDict作为基类，总比以普通的dict作为基类要来得方便

- 更倾向于从UserDict而不是dict继承的原因是,后者有时会在某些方法的实现上走一些捷径，导致我们不得不在它的子类中重写这些方法，但是UserDict就不会带来这些问题

- 另外一个值得注意的地方是，UserDict并不是dict的子类，但是UserDict有一个叫做data的属性，是dict的实例，这个属性实际上是UserDict最终存储数据的地方

- demo：StrKeyDict
    ````py
    import collections


    # StrKeyDict是对UserDict的扩展
    class StrKeyDict(collections.UserDict):
        
        def __missing__(self, key):
            if isinstance(key, str):
                raise KeyError(key)
            return self[str(key)]
        
        # contains比上一章的实现要简单,因为通过重写setitem,此时可以默认所有的key都是字符串
        def __contains__(self, key):
            return str(key) in self.data
        
        # setitem会把所有的键都转换为字符串
        def __setitem__(self, key, value):
            self.data[str(key)] = value
        
        # get方法与父类的get方法完全一致,所以不用再实现一遍

    ````

- 因为UserDict继承的是MutableMapping，所以StrKeyDict里剩下的那些映射类型的方法都是从UserDict、MutableMapping和Mapping这些超类继承而来的
---

### 3.7 不可变映射类型
---

- 从Python3.3开始，types模块中引入了一个封装类名为MappingProxyType
    - 如果给这个类一个映射，它会返回一个只读的映射视图
    - 虽然是个只读视图，但它是动态的
    - 这意味着如果对原映射作出了改动，我们可以通过这个视图观察到，但无法通过这个视图对原映射作出修改
    
- demo:用MappingProxyType来获取字典的只读实例mappingproxy
    ````py
    >>> from types import MappingProxyType
    >>> d = {1:'A'}
    >>> d_proxy = MappingProxyType(d)
    >>> d_proxy
    mappingproxy({1: 'A'})
    # d中的内容在d_proxy中可以看到
    >>> d_proxy[1]
    'A'
    # 但是通过d_proxy不能对它作出任何修改
    >>> d_proxy[2] = 'B'
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    TypeError: 'mappingproxy' object does not support item assignment
    >>> d[2] = 'B'
    # d_proxy是动态的，也就是说对d所做的任何改动都会反馈到它的上面
    >>> d_proxy
    mappingproxy({1: 'A', 2: 'B'})
    >>> d_proxy[2]
    'B'
    >>> 

    ````
---

### 3.8 集合论

- set和它的不可变姊妹类型frozenset直到Python2.3才首次以模块的形式出现,然后在Python2.6中它们升级为内置类型

- 集合的本质是许多唯一对象的聚集。因此，集合可以用来去重:
    ````py
    >>> l = ['spam', 'spam', 'eggs', 'spam']
    >>> set(l)
    {'spam', 'eggs'}
    >>> list(set(l))
    ['spam', 'eggs']
    ````

- 集合中的元素必须是可散列的，set类型本身是不可散列的，但是frozenset可以，因此可以创建一个包含不同frozenset的set

- 除了保证唯一性，集合还实现了许多基础的中缀运算符
    - `a | b` 返回它们的合集
    - `a & b` 得到的是交集
    - `a - b` 得到的是差集
---
#### 3.8.1 集合字面量

- 除了空集之外，集合的字面量—————{1}、{1, 2}，等等，看起来跟它的数学形式一模一样；如果是空集，那么必须写成set()形式

- 如果要创建一个空集，必须使用不带任何参数的构造方法set()。如果只是写成{}的形式，跟以前一样，你创建的其实是空字典

- 除了空集，集合的字符串表示形式总是以{...}的形式出现
    ````py
    >>> s = {1}
    >>> type(s)
    <class 'set'>
    >>> s
    {1}
    >>> s.pop()
    1
    >>> s
    set()
    >>> 
    ````

- 像{1, 2, 3}这样字面量句法相比于构造方法`set([1, 2, 3])`要更快且更易读
    - 针对{1, 2, 3}这样的字面量，Python会利用一个专门的叫做BUILD_SET的字节码来创建集合

- 由于Python没有针对frozenset的特殊字面量句法，我们只能采用构造函数
    ````py
    >>> frozenset(range(10))
    frozenset({0, 1, 2, 3, 4, 5, 6, 7, 8, 9})
    ````
---
#### 3.8.2 集合推导

- 示例
````py
# 从unicodedata模块中导入name函数，用以获得字符的名字
>>> from unicodedata import name
# 把编码在32-255之间的字符的名字里有"SIGN"的单词挑出来，放到一个集合中
>>> {chr(i) for i in range(32, 256) if 'SIGN' in name(chr(i), '')}
{'$', 'µ', '¬', '÷', '<', '§', '×', '=', '¢', '¶', '%', '+', '£', '±', '>', '#', '®', '©', '¤', '°', '¥'}
````
---
#### 3.8.3 集合的操作

### 3.9 dict和set的背后
---
#### 3.9.1 一个关于效率的实验

- 用in运算符在5个不同大小的haystack字典里搜索1000个元素所需要的时间

- 在5个不同大小的haystack里搜索1000个元素所需要的时间，haystack分别以字典、集合和列表的形式出现
---
#### 3.9.2 字典中的散列表

- 简介
    - 散列表其实是一个稀疏数组(总是有空白元素的数组称为稀疏数组)
    - 在一般的数据结构教材中，散列表里的单元叫做表元(bucket)
    - 在dict的散列表当中，每个键值对都占用一个表元
    - 每个表元都有两个部分，一个是对键的引用，另一个是对值的引用
    - 因为所有表元的大小一致，所以可以通过偏移量来读取某个表元

- Python会设法保证大概还有三分之一的表元是空的，所以在快要达到这个阈值的时候，原有的散列表会被复制到一个更大的空间中

- 散列值和相等性

    - 内置的hash()方法可以用在所有的内置类型对象
    
    - 如果是自定义对象调用hash()的话，实际上运行的是自定义的__hash__
    
    - 如果两个对象在比较的时候是相等的，那它们的散列值必须相等，否则散列表就不能正常运行了
        - 例如，如果 1 == 1.0 成立，那么hash(1) == hash(1.0)也必须成立
    
    - 为了让散列表能够胜任散列表索引这一角色，它们必须在索引空间中尽量分散开来
        - 最理想的状况下，越是相似但是不相等的对象，它们散列值的差别应该越大

    - 加盐
        - str、bytes和datetime对象的散列值计算有随机的"加盐"这一步
        - 所加盐值是Python进程内的一个常量，但是每次启动Python解释器都会生成一个不同的盐值
        - 随机盐值的加入是为了防止dos攻击而采取的一种安全措施

- 散列表算法

    - 算法的步骤
        1. 为了获取my_dict[search_key]背后的值，Python首先会调用hash(search_key)来计算search_key的散列值
        2. 然后把散列值的最低几位数字作为偏移量，在散列表里查找表元
        3. 若找到的表元为空，则抛出KeyError异常
        4. 若不是空的，则表元里会有一对found_key:found_value。这时检测search_key是否等于found_key，如果它们相等的话，就返回found_value
    
    - 另外
        - 在插入新值的时候，Python可能会按照散列表的拥挤程度来决定是否要重新分配内存为它扩容，这样做的目的是为了减少散列冲突发生的概率
---
#### 3.9.3 dict的实现及其导致的结果

1. 键必须是可散列的
    - 一个可散列的对象需要满足的条件
        1. 支持hash()函数，并且通过__hash__()方法得到的散列值是不变的
        2. 支持通过__eq__()方法来检测相等性
        3. 若 a==b 为真，则 hash(a) == hash(b) 也为真
    - 所有由用户自定义的对象默认都是可散列的，因为它们的散列值有id()来获取，而且它们是不相等的
    - 如果你实现了一个类的__eq__方法，并且希望它是可散列的，那么它一定要有个恰当的__hash__方法，保证在a==b为真的情况下hash(a)==hash(b)也必定为真

2. 字典在内存上的开销巨大
    - 由于字典使用了散列表，散列表必须是稀疏的，这导致它在空间上的效率低下
    - 在用户自定义的类型中，__slot__属性可以改变实例属性的存储方式，由dict变成tuple

3. 键查询很快
    - dict的实现是典型的空间换时间：字典有巨大的内存开销，但它们提供了无视数据量大小的快速访问————只要字典能被放入内存中

4. 键的次序取决于添加顺序
    - 这是因为当往dict里添加新键而又发生散列冲突的时候，新键可能会被安排存放到另一个位置
    - 但是即使顺序不同，内容相同的字典也被认为是相等的

5. 往字典里添加新键可能会改变原有键的顺序
    - 因为往字典中添加键可能会导致字典扩容，从而改变原有键的位置和顺序
    - 所以不要在对字典进行迭代的同时进行修改
---
#### 3.9.4 set的实现及其导致的结果

- set和frozenset的实现也依赖散列表，但在它们的散列表中存放的只有元素的引用

- 集合的特点
    1. 集合中的元素必须是可散列的
    2. 集合很消耗内存
    3. 可以很高效地判断元素是否存在于某个集合
    4. 元素的次序取决于被添加到集合里的次序
    5. 往集合中添加元素，可能会改变集合里已有元素的顺序
---


