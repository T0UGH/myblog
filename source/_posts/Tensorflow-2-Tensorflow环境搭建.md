---
title: '[Tensorflow][2]Tensorflow环境搭建'
date: 2019-03-10 14:35:36
tags:
    - 深度学习
    - Tensorflow
categories:
    - Tensorflow
---

## 第2章 TensorFlow环境搭建
---
### 2.1 TensorFlow的主要依赖包
---
#### 2.1.1 Protoval Buffer

- Protocal Buffer是谷歌开发的处理结构化数据的工具

- 什么是结构化数据
    - 例如：用户信息包括用户的名字、ID和Email
    ````
    name:张三
    id:12345
    email:zhangsan@abc.com
    ````
    上面的用户信息就是一个结构化的数据

- 结构化数据处理
    - 背景：要将这些结构化的用户信息持久化或进行网络传输时，就需要先将它们序列化。
    - 序列化：所谓序列化，是将结构化的数据变成数据流的格式，简单地说就是变成一个字符串
    - 定义：将结构化的数据序列化，并从序列化之后的数据流中还原出原来的结构化数据，统称为结构化数据处理

- 常见的结构化数据处理工具
    - XML
    - JSON
    - Protocal Buffer

- Protocal Buffer
    - 与XML和JSON格式的区别
        - Protocal Buffer序列化之后得到的数据不是可读的字符串，而是二进制流。
        - 使用Protocal Buffer时需要先定义数据的格式(schema)。还原一个序列化之后的数据将需要使用到这个定义好的数据格式  
        例如
        ````
        message user{
            optional string name = 1;
            required int32 id = 2;
            repeated string email = 3;
        }
        ````
    - 基本语法
        - Protocal Buffer定义数据格式的文件一般保存在.proto文件中
        - 每个message代表了一类结构化的数据
        - message里面定义了每个属性的类型和名字
        - 属性的类型可以是像布尔型、整数型、实数型、字符型这样的基本类型，也可以使另外一个message
        - 也需要定义一个属性是必需的(required)、可选的(optional)、或者可重复的(repeated)
            - required：这个message的所有实例都需要这个属性
            - optional：这个属性的取值可以为空
            - repeated：这个属性的取值可以使一个列表

---
#### 2.1.2 Bazel

- 简介：Bazel是谷歌开源的自动化构建工具

- 项目空间(workspace)
    - 一个项目空间可以简单地理解为一个文件夹
    - 在这个文件夹中包含了编译一个软件所需要的源代码以及输出编译结果的软连接(symbolic link)地址

- WORKSPACE文件
    - 一个项目空间所对应的文件夹是这个项目的根目录，在这个根目录中需要一个WORKSPACE文件，此文件定义了对外部资源的依赖关系

- BUILD文件
    - 在一个项目空间中,Bazel通过BUILD文件来找到需要编译的目标
    - Bazel对Python支持的编译方式
        1. py_binary：将Python程序编译为可执行文件
        2. py_test：编译Python测试程序
        3. py_library：编译为库函数供其他py_binary或py_test调用
    - 例子
    ````py
    py_library(
        name="hello_lib",
        srcs=[
            "hello_lib.py",
        ]
    )
    py_binary(
        name="hello_main",
        srcs=[
            "hello_main.py",
        ]
        deps=[
            ":hello_lib",
        ],
    )
    ````
    - 其他
        - BUILD文件由一系列编译目标组成
        - 定义编译目标的先后顺序不会影响编译的结果
        - 在每个编译目标的第一行要指定辨析方式
        - 在每个编译目标的主体中需要给出编译的具体信息，例如：names、srcs、deps等