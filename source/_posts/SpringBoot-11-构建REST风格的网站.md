---
title: '[SpringBoot][11][构建REST风格的网站]'
date: 2020-02-09 12:03:38
tags:
    - REST
    - 数据库事务
categories:
    - SpringBoot
---
## 第 11 章 构建REST风格网站

在`HTTP`协议发展的过程中，提出了很多的规则，但是这些规则有些烦琐，于是又提出了一种风格约定，它便是`REST`风格。实际上严格地说它不是一种标准，而是一种风格。在现今互联网的世界中这种风格己经被广泛使用起来了。尤其是现今流行的微服务中，这样的风格甚至被推荐为各个微服务系统之间用于交互的方式。

首先在`REST`风格中，每一个资源都只是对应着一个网址，而一个代表资源的网址应该是一个名词，且不存在动词，这代表对一个资源的操作。在这样的风格下对于简易参数则尽量通过网址进行传递。例如，要获取`id`为`l`的用户的`URL`可能就设计`http//localhost:8080/user/1`

### 11.1 REST简述

#### 11.1.1 REST名词解释

`REST`按其英文名称(`Representational State Transfer`)可翻译为表现层状态转换。首先需要有资源才能表现，所以第一个名词是"资源"。有了资源也要根据需要以合适的形式表现资源，这就是第二个名词是"表现层"。最后是资源可以被新增、修改、删除等，也就是第三个名词“状态转换”
- 资源: 它可以是系统权限用户、角色和菜单等，也可以是一些媒体类型，如文本、图片、歌曲，总之它就是一个具体存在的对象。每个资源对应一个独一无二的`URI`。在`REST`中，`URI`也可以称为端点(`EndPoint`)。
- 表现层: 有了资源还需要确定如何表现这个资源。例如，一个用户可以使用`JSON`、`XML`或者其他的形式表现出来
- 状态转换: 现实中资源并不是一成不变的， 它是一个变化的过程，一个资源可以经历创建(`create`)、访问)(`visit`) 、修改(`update`)和删除(`delete`)的过程

综上我们可以总结`REST`风格架构的特点
- 服务器存在一系列的资源，每一个资源通过单独唯一的`URI`进行标识
- 客户端和服务器之间可以相互传递资源，而资源会以某种表现层得以展示
- 客户端通过`HTTP`协议所定义的动作对资源进行操作，以实现资源的状态转换

#### 11.1.2 HTTP的动作

可以用`HTTP`请求的类型，来代表对资源的`CRUD`的行为
- `GET`: `READ`，访问服务器资源
- `POST`: `CREATE`，在服务器创建新的资源
- `PUT`: `UPDATE`，修改服务器已经存在的资源，使用`PUT`时需要把资源的全部属性一并提交
- `PATCH`: `UPDATE`，修改服务器已经存在的资源，使用`PATCH`时只需要将部分资源属性提交
- `DELETE`: `DELETE`，从服务器将资源删除

下面举几个`REST`风格的例子
````
# 获取用户信息，1代表用户编号
GET /user/1
# 查询多个用户信息
GET /users/{userName}/{note}
# POST 创建用户
POST /user/{userName}/{sex}/{note}
# 修改用户全部属性
PUT /user/{id}/{userName}/{sex}/{note}
# 修改用户名称
PATCH /user/{id}/{userName}
````
在`URI`中并没有出现动词，而对于参数主要通过`URI`设计去获取。对于参数数量超过5个的可以考虑使用传递`JSON`的方式来传递参数

#### 11.1.3 REST风格的一些误区

1. REST风格的URI中不存在动词，例如`GET /user/get/1`，应改为`GET /user/1`

2. URI中不应该加入版本号，例如下面`GET /v1/user/1`，如果存在版本号应设置在请求头中

3. 类似这种`PUT users?userName=user_name&note=note`是不推荐使用的，应改为`PUT users/{userName}/{note}`

### 11.2 使用SpringMVC开发REST风格端点

`Spring`对`REST`风格的支持是基于`SpringMVC`设计基础上的，在`Spring 4.3`之后则有更多的注解引入使得`REST`风格的开发更为便捷。

#### 11.2.1 SpringMVC整合REST

只要把`URI`设计为符合`REST`风格规范，那么显然就己经满足`REST`风格了。不过为了更为便捷地支持`REST`风格的开发，`Spring 4.3`之后除了`@RequestMapping`外，还可以使用以下5个注解，这5个注解主要是针对`HTTP`的动作而言的，通过它们就能够有效地支持`REST`风格的规范
- `@GetMapping`: 对应`HTTP`的`GET`请求，获取资源
- `@PostMapping`: 对应`HTTP`的`POST`请求，创建资源
- `@PutMapping`: 对应`HTTP`的`PUT`请求，提交所有资源属性以修改资源
- `@PatchMapping`: 对应`HTTP`的`PATCH`请求，提交资源部分修改的属性
- `@DeleteMapping`: 对应`HTTP`的`DELETE`请求，删除服务器端的资源

在`REST`风格的设计中，如果是简单的参数，往往会通过`URL`直接传递，在`SpringMVC`可以使用注解`@PathVariable`进行获取，对于那些复杂的参数，可以考虑使用请求体`JSON`的方式提交给服务器，这样就可以使用注解`@RequestBody`将`JSON`数据集转换为`Java`对象。

在现今的开发中，数据转化为`JSON`是最常见的方式，这个时候可以考虑使用注解`@ResponseBody`，这样`SpringMVC`就会通过`MappingJackson2HttpMessageConverter`最终将数据转换为`JSON`数据集，而在`SpringMVC`对`REST`风格的设计中，甚至可以使用注解`@RestController`让整个控制器都默认转换为`JSON`数据集。

#### 11.2.2 使用Spring开发REST风格端点

我们可以使用`POST`动作来创建资源
````java
@PostMapping("/user")
@ResponseBody
public User insertUser(@RequestBody User user){
    return userService.insertUser(user);
}
````
- `@PostMapping`表示采用`POST`动作提交用户信息
- `@ReguestBody`代表接收的是一个`JSON`数据集参数
- `@ResponseBody`代表会将函数的返回值转化为`JSON`格式传递给前端

接下来就是使用`GET`动作来获取对象了
````java
@GetMapping(value="/user/{id}")
@ResponseBody
public User getUser(@PathVariable("id")Long id){
    return userService.getUser(id);
}
````
- 采用注解`@GetMapping`声明`HTTP`的`GET`请求，并且把参数编号(`id`)以`URI`的形式传递，这符合了`REST`风格的要求
- 在`getUser`方法中使用了注解`@PathVariable`从`URI`中获取参数
- `@ResponseBody`代表会将函数的返回值转化为`JSON`格式传递给前端

更新和删除操作也与上面的例子类似，这里略

#### 11.2.3 使用@RestController

因为现在前后端分离，所以使用`JSON`作为前后端交互已经十分普遍。如果每一个方法都加入`@ResponseBody`才能将数据模型转换为`JSON`，显然有些冗余。`SpringMVC`还存在一个注解`@RestController`，它可以修饰控制器类，将类中的方法返回值转化为`JSON`数据

````java
@RestController
public class UserController{
    @GetMapping(value="/user/{id}")
    public User getUser(@PathVariable("id")Long id){
        return userService.getUser(id);
    }
}
````

#### 11.2.4 渲染结果

在`@RequestMapping`、`GetMapping`等注解中还存在`consumes`和`produces`两个属性。其中`consumes`
代表的是限制该方法接收什么类型的请求体(body), `produces`代表的是限定返回的媒体类型，仅当`request`请求头中的(`Accept`)类型中包含该指定类型才返回。

假如我们想要某个方法返回一个字符串，而不是JSON的话，可以采用如下方法
````java
@GetMapping(value="/user/name/{id}", produces=MediaType.TEXT_PLAIN_VALUE)
public String getUserName(@PathVariable("id") Long id){
    User user = userService.getUser(id);
    return user.getUserName();
}
````
- 对于`getUserName`方法，因为`@GetMapping`的属性`produces`声明为普通文本类型，也就是修改了原有`@RestController`默认的`JSON`类型，同样结果也会被`SpringMVC`自身注册好的`StringHttpMessageConverter`拦截， 这样就可以转变为一个简单的字符串。

#### 11.2.5 处理HTTP状态码、异常和响应头

当发生资源找不到或者处理逻辑发生异常时， 需要考虑的是返回给客户端的`HTTP`状态码和错误消息的问题。为了简化这些开发，`Spring`提供了实体封装类`ResponseEntity`和注解`@ResponseStatus`。`ResponseEnti ty`可以有效封装错误消息和状态码，通过`@ResponseStatus`可以配置指定的响应码给客户端。

下面我们修改插入用户的方法，将状态码修改为201，并且插入响应头的属性来标识这次请求的结果
````java
@PostMapping(value = "/user2/entity")
public ResponseEntity<UserVo> insertUserEntity(
        @RequestBody UserVo userVo) {
    User user = this.changeToPo(userVo);
    userService.insertUser(user);
    UserVo result = this.changeToVo(user);
    HttpHeaders headers = new HttpHeaders();
    String success = 
        (result == null || result.getId() == null) ? "false" : "true";
    // 设置响应头，比较常用的方式
    headers.add("success", success);
    // 返回创建成功的状态码
    return new ResponseEntity<UserVo>(result, headers, HttpStatus.CREATED);
}
@PostMapping(value = "/user2/annotation")
// 指定状态码为201（资源已经创建）
@ResponseStatus(HttpStatus.CREATED)
public UserVo insertUserAnnotation(@RequestBody UserVo userVo) {
    User user = this.changeToPo(userVo);
    userService.insertUser(user);
    UserVo result = this.changeToVo(user);
    return result;
}
````
- `insertUserEntity`方法中定义返回为一个`ResponseEntity<User>`的对象，这里还生成了响应头( `HttpHeaders`对象)，并且添加了属性`success`来表示请求是否成功，在最后返回的时刻生成了一个`ResponseEntity<User>`对象，然后将查询到的用户对象和响应头捆绑上，并且指定状态码为201(创建资源成功)
- 在`insertUserAnnotation`方法上则使用了`@ResponseStatus`注解将`HTTP`的响应码标注为201(创建资源成功)，所以在方法正常返回时`Spring`就会将响应码设置为`20l`

但是有时候会出现一些异常，例如，按照`id`查找用户，可能查找不到数据，这个时候就不能以正常返回去处理了，又或者在执行的过程中产生了异常，这也是需要我们进行处理的。通过`@ControllerAdvice`和`@ExceptionHandler`注解可以通过控制器通知的方式对异常情况进行处理

首先我们自定义一个查找失败异常
````java
public class NotFoundException extends RuntimeException {
	private static final long serialVersionUID = 1L;
	// 异常编码
	private Long code;
	// 异常自定义信息
	private String customMsg;

	public NotFoundException() {
	}

	public NotFoundException(Long code, String customMsg) {
		super();
		this.code = code;
		this.customMsg = customMsg;
	}
    //getter and setter
}
````

然后自定义一个控制器通知，来自定义异常的处理
````java
//控制器通知
@ControllerAdvice(
		// 指定拦截包的控制器
		basePackages = { "com.springboot.chapter11.controller.*" },
		// 限定被标注为@Controller或者@RestController的类才被拦截
		annotations = { Controller.class, RestController.class })
public class VoControllerAdvice {
	// 异常处理，可以定义异常类型进行拦截处理
	@ExceptionHandler(value=NotFoundException.class)
	// 以JSON表达方式响应
	@ResponseBody
	// 定义为服务器错误状态码
	@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
	public Map<String, Object> exception(HttpServletRequest request, NotFoundException ex) {
		Map<String, Object> msgMap = new HashMap<>();
		// 获取异常信息
		msgMap.put("code", ex.getCode());
		msgMap.put("message", ex.getCustomMsg());
		return msgMap;
	}
}
````
- 这里使用了`@ControllerAdvice`来标注类，说明在定义一个控制器通知。
- `basePackages`配置了它所拦截的包，`annotations`限定了拦截的那些被标注为注解`@Controller`和`@RestController`的控制器，
- 这里的`@ExceptionHandler`定义了拦截`NotFoundException`的异常
- `@ResponseBody`定义了响应的信息以`JSON`格式表达
- `@ResponseStatus`定义了状态码为`500`(服务器内部错误)，这样就会把这个状态码传达给请求者

最后我们让前文定义的`getUser`方法抛出异常，这样完成了当发生资源找不到或者处理逻辑发生异常这种情况下的处理逻辑
````java
@GetMapping(value="/user/{id}")
@ResponseBody
public User getUser(@PathVariable("id")Long id){
    User user = userService.getUser(id);
    if(user == null){
        throw new NotFoundException(1L, "找不到用户["+id+"]信息")
    }
    return userService.getUser(id);
}
````

这样当发生异常时，就可以返回`500`状态码和异常信息字符串了
