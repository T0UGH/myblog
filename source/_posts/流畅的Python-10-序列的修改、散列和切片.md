---
title: '[流畅的Python][9][序列的修改、散列和切片]'
date: 2020-02-01 20:49:56
tags:
    - Python
categories:
    - 流畅的Python
---

## 第10章 序列的修改、散列和切片

> 不要检查它是不是鸭子，它的叫声像不像鸭子、它的走路姿势像不像鸭子，等等。具体检查什么取决于你想使用语言的哪些行为

- 在大量代码之间，我们将穿插讨论一个概念：把协议当作正式接口。我们将说明协议与鸭子类型之间的关系，以及对自定义类型的实际影响

### 10.1 Vector类：用户定义的序列类型

- 我们将使用组合模式实现`Vector`类，而不使用继承

- 向量的分量存储在浮点数数组中，而且还将实现不可变扁平序列所需的方法

---

### 10.2 Vector类第1版：与Vector2d类兼容

- 示例：第 1 版 `Vector` 类的实现代码
    ````py
    from array import array
    import reprlib
    import math


    class Vector:
        typecode = 'd'

        # self._components是“受保护的”实例属性，把Vector的分量保存在一个数组中
        def __init__(self, components):
            self._components = array(self.typecode, components)

        # 为了迭代，我们使用self._components构建一个迭代器
        def __iter__(self):
            return iter(self._components)

        def __repr__(self):
            # 使用reprlib.repr()函数获取self._components的有限长度表示形式(如array('d', [0.0, 1.0, 2.0, 3.0, ...]))
            components = reprlib.repr(self._components)
            # 把字符串插入Vector的构造方法调用之前，去掉前面的 array('d' 和后面的 )
            components = components[components.find('('):-1]
            return 'Vector({})'.format(components)

        def __str__(self):
            return str(tuple(self))

        def __bytes__(self):
            # 直接使用self._components构建bytes对象
            return bytes([ord(self.typecode)]) + bytes(self._components)

        def __eq__(self, other):
            return tuple(self) == tuple(other)

        def __abs__(self):
            # 先计算各分量的平方之和，然后再使用sqrt方法开平方
            return math.sqrt(sum(x * x for x in self))

        def __bool__(self):
            return bool(abs(self))

        @classmethod
        def frombytes(cls, octets):
            typecode = chr(octets[0])
            memv = memoryview(octets[1:]).cast(typecode)
            return cls(memv)
    ````

- 对 `repr()`函数的理解
    - 调用`repr()`函数的目的是调试，因此绝对不能抛出异常
    - 如果`__repr__`方法的实现有问题，那么必须处理，尽量输出有用的内容，让用户能够识别目标对象

- 使用`reprlib.repr()`的方式
    - 这个函数用于生成大型结构或递归结构的安全表示形式
    - 它会限制输出字符串的长度，用'...'表示截断的部分

- 示例：测试 `Vector.__init__` 和 `Vector.__repr__`
````py
>>> Vector([3.1, 4.2])
Vector([3.1, 4.2])
>>> Vector([3, 4, 5])
Vector([3, 4, 5])
>>> Vector(range(10))
Vector([0.0, 1.0, 2.0, 3.0, 4.0, ...])
````

---

### 10.3 协议和鸭子类型

- 协议
    - 在Python中创建功能完善的序列类型无需使用**继承**，只需**实现符合序列协议的方法**
    - 在面向对象编程中，**协议**是非正式的**接口**，只在文档中定义，在代码中不定义
    - 例如，Python的序列协议只需要 `__len__` 和 `__getitem__`两个方法。任何类，只要使用标准的签名和语义实现了这两个方法，就能用在任何期待序列的地方


- 鸭子类型
    - 不要检查它是不是鸭子，它的叫声像不像鸭子、它的走路姿势像不像鸭子，等等。具体检查什么取决于你想使用语言的哪些行为
    - 我们说它是序列，因为它的行为像序列，这才是重点

- 另外
    - 协议是非正式的，没有强制力。因此如果你知道类的具体使用场景，通常只需要实现一个协议的部分
    - 例如，为了支持迭代，只需要实现 `__getitem__` 方法，没必要提供 `__len__` 方法

---

### 10.4 Vector类第2版：可切片的序列

