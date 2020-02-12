---
title: '[SpringBoot][8][文档数据库MongoDB]'
date: 2020-02-12 12:29:33
tags:
    - SpringBoot
    - MongoDB
categories:
    - SpringBoot
---
## 第 8 章 文档数据库MongoDB

对于那些需要缓存而且经常需要统计、分析和查询的数据，`Redis`这样简单的`NoSQL`显然就不是那么便捷了， 这时另外一个`NoSQL`就派上用场了，它就是本章的主题`MongoDB`。对于那些需要统计、按条件查询和分析的数
据，它提供了支持，它可以说是一个最接近于关系数据库的`NoSQL`。

`MongoDB`是由`C++`语言编写的一种`NoSQL`，是一个基于分布式文件存储的开源数据库系统。在负载高时可以添加更多的节点，以保证服务器性能，`MongoDB`的目的是为`Web`应用提供可扩展的高性能数据存储解决方案。`MongoDB`将数据存储为一个文档，数据结构由键值(`key-value`)对组成。这里的`MongoDB`文档类似于`JSON` 数据集，所以很容易转化成为`JavaPOJO`对象或者`JavaScript`对象，这些字段值还可以包含其他文档、数组及文档数组。

### 8.1 配置MongoDB

`SpringBoot`提供了`MongoDB`的配置，其默认的可配置项如下所示
````
spring.data.mongodb.host=192.168.11.131 #MongoDB服务器
spring.data.mongodb.username=spring #MongoDB服务器用户名
spring.data.mongodb.password=123456 #MongoDB服务器密码
spring.data.mongodb.port=27017 #MongoDB服务器端口
spring.data.mongodb.database=springboot #数据库名称
spring.data.mongodb.repository.type=auto #是否启用MongoDB关于JPA规范的编程
spring.data.mongodb.field-naming-strategy=xxx #使用字段名策略
````

### 8.2 使用MongoTemplate实例

`spring-data-mongodb`主要是通过`MongoTemplate`进行操作数据。`SpringBoot`会根据配置自动生成`MongoTemplate`对象来操作数据，下面举个例子来说明

首先创建`POJO`
````java
package com.springboot.chapter8.pojo;

import java.io.Serializable;
import java.util.List;
import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;

// 标识为MongoDB文档
@Document
public class User implements Serializable {
	private static final long serialVersionUID = -7895435231819517614L;
	// MongoDB文档编号，主键
	@Id
	private Long id;
	// 在MongoDB中使用user_name保存属性
	@Field("user_name")
	private String userName = null;

	private String note = null;
	// 角色列表
    @DBRef
	private List<Role> roles = null;
    //setter and getter
}
````
- 这个文档被标记为`@Document`，这说明它将作为`MongoDB`的文档存在。注解`@id`则将对应的字段设置为主键。`@Field`可以将属性`userName`与`MongoDB`中的`user_name`属性对应起来。然后角色列表，如果只想保存其引用，可以使用`@DBRef`进行标注

然后编写一个`Service`来看看`MongoTemplate`如何操作`MongoDB`文档
````java
@Service
public class UserServiceImpl implements UserService {

	// 注入MongoTemplate对象
	@Autowired
	private MongoTemplate mongoTmpl = null;

	@Override
	public User getUser(Long id) {
		return mongoTmpl.findById(id, User.class);
		// 如果只需要获取第一个也可以采用如下查询方法
		// Criteria criteriaId = Criteria.where("id").is(id);
		// Query queryId = Query.query(criteriaId);
		// return mongoTmpl.findOne(queryId, User.class);
	}

	@Override
	public List<User> findUser(String userName, String note, int skip, int limit) {
		// 将用户名称和备注设置为模糊查询准则
		Criteria criteria = Criteria.where("user_name").regex(userName).and("note").regex(note);
		// 构建查询条件,并设置分页跳过前skip个，至多返回limit个
		Query query = Query.query(criteria).limit(limit).skip(skip);
		// 执行
		List<User> userList = mongoTmpl.find(query, User.class);
		return userList;
	}

	@Override
	public void saveUser(User user) {
		// 使用名称为user文档保存用户信息
		mongoTmpl.save(user, "user");
		// 如果文档采用类名首字符小写，则可以这样保存
		// mongoTmpl.save(user);
	}

	@Override
	public DeleteResult deleteUser(Long id) {
		// 构建id相等的条件
		Criteria criteriaId = Criteria.where("id").is(id);
		// 查询对象
		Query queryId = Query.query(criteriaId);
		// 删除用户
		DeleteResult result = mongoTmpl.remove(queryId, User.class);
		return result;
	}

	@Override
	public UpdateResult updateUser(Long id, String userName, String note) {
		// 确定要更新的对象
		Criteria criteriaId = Criteria.where("id").is(id);
		Query query = Query.query(criteriaId);
		// 定义更新对象，后续可变化的字符串代表排除在外的属性
		Update update = Update.update("user_name", userName);
		update.set("note", note);
		// 更新单个对象
		UpdateResult result = mongoTmpl.updateFirst(query, update, User.class);
		// 更新多个对象
		// UpdateResult result2 = mongoTmpl.updateMulti(query, update, User.class);
		return result;
	}

}
````
- 注意: 在`MongoDB`中的`save`，若已存在`id`相同的对象，那么更新其属性；如果是已经存在的对象，则它只是对对象进行更新