---
title: YAML学习
date: 2019-03-03 13:57:29
tags:
  - YAML
categories:
  - other
---
## YAML学习

### 1 简介

YAML语言的设计目标，就是方便人类书写。它实质上是一种通用的数据串行化格式

#### 1.1 YAML的基本语法规则

1. 大小写敏感

2. 使用缩进表示层级关系

3. 缩进时不允许使用tab键，只允许使用空格

4. 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可

5. #表示注释，从这个字符一直到行尾，都会被解释器忽略

#### 1.2 YAML支持的数据结构有三种

1. 对象: 键值对的集合, 又称为映射(mapping)/哈希(hashes)/字典(dictionary)

2. 数组: 一组按次序排列的值, 又称为序列(sequence)/列表(list)

3. 纯量(scalars): 单个的, 不可再分的值

### 2 对象

对象是一组键值对, 使用冒号结构表示
````yaml
animal: pets
````

转为JavaScript如下
````js
{animal: 'pets'}
````

### 3 数组

一组连词线开头的行，构成一个数组

````yaml
- Cat
- Dog
- Goldfish
````

转为JavaScript如下
````js
['Cat', 'Dog', 'Goldfish']
````

数据结构的子成员是一个数组，则可以在该项下面缩进一个空格
````yaml
-
 - Cat
 - Dog
 - Goldfish
````

转为 JavaScript 如下
````js
[ [ 'Cat', 'Dog', 'Goldfish' ] ]
````

数组也可以采用行内表示法
````yaml
animal: [Cat, Dog]
````

转为 JavaScript 如下
````js
{ animal: [ 'Cat', 'Dog' ] }
````

### 4 复合结构

对象和数组可以结合使用，形成复合结构。
````yaml
languages:
 - Ruby
 - Perl
 - Python 
websites:
 YAML: yaml.org 
 Ruby: ruby-lang.org 
 Python: python.org 
 Perl: use.perl.org 
````

转为 JavaScript 如下
````js
{ languages: [ 'Ruby', 'Perl', 'Python' ],
  websites: 
   { YAML: 'yaml.org',
     Ruby: 'ruby-lang.org',
     Python: 'python.org',
     Perl: 'use.perl.org' } }
````

### 5 纯量

纯量是最基本的、不可再分的值。以下数据类型都属于JavaScript纯量

- 字符串
- 布尔值
- 整数
- 浮点数
- Null
- 时间
- 日期

数值直接以字面量的形式表示。

````yaml
number: 12.30
````

转为 JavaScript 如下

````js
{ number: 12.30 }
````

布尔值用true和false表示

````yaml
isSet: true
````

转为 JavaScript 如下
````js
{ isSet: true }
````

`null`用`~`表示
````yaml
parent: ~ 
````

转为 JavaScript 如下
````js
{ parent: null }
````

时间采用 ISO8601 格式
````yaml
start_time: 2001-12-14t21:59:43.10-05:00 
````

转为 JavaScript 如下
````js
{ iso8601: new Date('2001-12-14t21:59:43.10-05:00') }
````

日期采用复合 iso8601 格式的年、月、日表示
````yaml
date: 1976-07-31
````

转为JavaScript如下
````js
{ date: new Date('1976-07-31') }
````

YAML 允许使用两个感叹号，强制转换数据类型
````yaml
e: !!str 123
f: !!str true
````

转为 JavaScript 如下
````js
{ e: '123', f: 'true' }
````

### 6 字符串

字符串是最常见，也是最复杂的一种数据类型

字符串默认不使用引号表示

````yaml
str: 这是一行字符串
````

如果字符串之中包含空格或特殊字符，需要放在引号之中
````yaml
str: '内容： 字符串'
````

### 7 引用

锚点`&`和别名`*`，可以用来引用。

````yaml
defaults: &defaults
  adapter:  postgres
  host:     localhost

development:
  database: myapp_development
  <<: *defaults

test:
  database: myapp_test
  <<: *defaults
````

等同于下面的代码
````yaml
defaults:
  adapter:  postgres
  host:     localhost

development:
  database: myapp_development
  adapter:  postgres
  host:     localhost

test:
  database: myapp_test
  adapter:  postgres
  host:     localhost
````
`&`用来建立锚点（`defaults`），`<<`表示合并到当前数据，`*`用来引用锚点

下面是另一个例子
````yaml
- &showell Steve 
- Clark 
- Brian 
- Oren 
- *showell 
````

转为 JavaScript 代码如下
````js
[ 'Steve', 'Clark', 'Brian', 'Oren', 'Steve' ]
````

![](YAML学习/IBM_5100.jpg)

![](YAML学习/zhizhi.jpg)


