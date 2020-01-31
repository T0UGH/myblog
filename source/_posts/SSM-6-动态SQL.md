---
title: '[SSM][6][动态SQL]'
date: 2020-01-31 11:10:49
tags:
    - MyBatis
categories:
    - SSM
---

## 第6章 动态SQL

如果使用`JDBC`或者类似于`Hibernate`的其他框架，很多时候要根据需要去拼装`SQL`,这是一个麻烦的事情。因为某些查询需要许多条件，比如查询角色，可以根据角色名称或者备注等信息查询，当不输入名称时使用名称作条件就不合适了。

通常使用其他框架需要大量的`Java`代码进行判断，可读性比较差，而`MyBatis`提供对`SQL`语句动态的组装能力，使用`XML`的几个简单的元素，便能完成动态`SQL`的功能。大量的判断都可以在`MyBatis`的映射`XML`里面配置，以达到许多需要大量代码才能实现的功能，大大减少了代码量，这体现了`MyBatis`的灵活、高度可配置性和可维护性。

### 6.1 概述

MyBatis的动态SQL包括以下一个元素

|元素|作用|备注|
|--|--|--|
|if|判断语句|单条件分支判断|
|choose(when, otherwise)|相当于JAVA中的switch和case语句|多条件分支判断|
|trim(where, set)|辅助元素，用于处理特定的SQL组装问题，比如去掉多余的and、or等|用于处理SQL拼装的问题|
|foreach|循环语句|在in语句等列举条件常用|

### 6.2 if元素

`if`元素相当于`Java`中的`if`语句，它常与`test`属性联合使用

下面用一个场景来说明，根据角色名称(`roleName`)来查找角色，如果`roleName`不为空，则采取构造对`roleName`的模糊查询，否则不要去构造这个条件

````xml
<select id="findRoles" parameterType="string" resultMap="roleResultMap">
    select role_no, role_name, note from t_role where 1 = 1
    <if test="roleName != null and roleName != ''">
        and role_name like concat('%', #{roleName}, '%')
    </if>
</select>
````

### 6.3 choose、when、otherwise元素

`choose、when、otherwise`元素相当于`java`中的`switch...case...default..`

下面用一个场景来说明
- 如果角色编号(`roleNo`)不为空，则只用角色编号作为条件查询。
- 当角色编号为空，而角色名称不为空，则用角色名称作为条件进行模糊查询。
- 当角色编号和角色名称都为空，则要求角色备注不为空。

````xml
<select id="findRoles" parameterType="role" resultMap="roleResultMap">
    select role_no, role_name, note from t_role
    where 1=1
    <choose>
        <when test="roleNo != null and roleNo != ''">
            and role_no = #{roleNo}
        </when>
        <when test="roleName != null and roleName !=''">
            and role_name like concat('%', #{roleName}, '%')
        </when>
        <otherwise>
            and note is not null
        </otherwise>
    </choose>
</select>
````

### 6.4 trim、where、set元素


#### 6.4.1 where元素

在6.3节的`SQL`语句上的动态元素的`SQL`中都加入了一个条件`1=1`，如果没有加入这个条件，那么可能就变为了这样一条错误的语句：
````xml
select role_no, role_name, note from t_role where and role_name like concat('%', #{roleName}, '%')
````

我们可以使用`where`元素来避免加入`1=1`这个条件

````xml
<select id="findRoles" parameterType="role" resultMap="roleResultMap">
    select role_no, role_name, note from t_role
    <where>
        <if test="roleName != null and roleName !=''">
            and role_name like concat('%', #{roleName}, '%')
        </if>
        <if test="note != null and note !=''">
            and note like concat('%', #{note}, '%')
        </if>
    </where>
</select>
````

当`where`元素里面的条件成立时，才会加入`where`这个`SQL`关键字到组装的`SQL`里面，否则就不加入

#### 6.4.2 trim元素

有时候要去掉的是一些特殊的`SQL`语法，比如常见的`and`、`or`。而使用`trim`元素也可以达到预期效果

````xml
<select id="findRoles" parameterType="string" resultMap="roleResultMap">
    select role_no, role_name, note from t_role
    <trim prefix="where" prefixOverrides="and">
        <if test="roleName != null and roleName !=''">
            and role_name like concat('%', #{roleName}, '%')
        </if>
    </trim>
</select>
````
- `trim`代表要去掉一些特殊的字符串
- `prefix`代表的是语句的前缀，而`prefixOverrides`代表的是要去掉哪种字符串

#### 6.4.3 set元素

在`Hibernate`中常常因为要更新某一对象，而发送所有的字段给持久对象，而现实中的场景是，只想更新某一个字段。在`MyBatis`中， 常常可以使用`set`元素来避免这样的问题，比如要更新一个角色的数据
````xml
<update id="updateRole" parameterType="role">
    update t_role
    <set>
        <if test="roleName != null and roleName !=''">
            role_name = #{roleName},
        </if>
        <if test="note != null and note != ''">
            note = #{note}
        </if>
    </set>
    where role_no = #{roleNo}
</update>
````

### 6.5 foreach元素

`foreach`元素是一个循环语旬，它的作用是遍历集合，它能够很好地支持数组和`List`、`Set`接口的集合，对此提供遍历功能。它往往用于`SQL`中的`in`关键字。

````xml
<select id="findUserBySex" resultType="user">
    select * from t role where role no in
    <foreach item="roleNo" index="index" collection="roleNoList" open="(" separator="," close=")">
        #{roleNo}
    </foreach>
</select>
````
- `collection`配置的`roleNoList`是传递进来的参数名称，它可以是一个数组、`List`、`Set`等集合
- `index`配置的是循环中当前的元素
- `index`配置的是当前元素在集合的位置下标
- `open`和`close`配置的是以什么符号将这些集合元素包装起来
- `separator`是各个元素的间隔符

在`SQL`中常常用到`in`语句，但是对于大量数据的`in`语句要特别注意，因为它会消耗大量的性能

### 6.6 用test的属性判断字符串

`test`用于条件判断语句，作用相当于判断真假

### 6.7 bind元素

在进行模糊查询时，如果是`MySQL`数据库，常常用到的是一个`concat`，它用`%`和参数相连。然而在`Oracle`数据库则没有，`Oracle`数据库用连接符号`||`，这样`SQL`就需要提供两种形式去实现。但是有了`bind`元素，就不必使用数据库的语言，而是使用`MyBatis`的动态`SQL`即可完成。

````xml
<select id="findRole" parameterType="string" resultType="com.bean.RoleBean">
    <bind name="pattern" value="'%' + _parameter + '%'">
    SELECT id, role_name as roleName, create_date as createDate, end_date as endFlag, note FROM t_role
    WHERE role_name like #{pattern}
</select>
````

这里的`_parameter`代表的是传递进来的参数，它和通配符(`%`)连接后赋给了`pattern`,然后就可以在`select`语句中使用这个变量进行模糊查询了。无论是`MySQL`还是`Oracle`都可以使用这样的语旬，提高了代码的可移植性。
