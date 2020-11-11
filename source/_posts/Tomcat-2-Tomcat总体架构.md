---
title: '[Tomcat][2][Tomcat总体架构]'
date: 2020-11-11 17:12:00
tags:
    - Tomcat
categories:
    - Tomcat
---
## 第2章 Tomcat总体架构



作为一款知名的轻量级应用服务器， Tomcat的**架构设计**（如**生命周期管理**、**可扩展的容器组件设计**、**类加载方式**）可以为我们的服务器中间件设计，甚至是应用系统组件设计提供**非常好的借鉴意义**。



本章主要包含如下几个部分

- Tomcat**总体架构设计**及 Tomcat**各组件的概念**。
- Tomcat**启动及请求处理过程**。
- Tomcat的**类加载器**。



### 2.1 总体设计



为了使读者能更深刻地理解 Tomcat的相关组件概念，我们将采用一种启发式的讲解方式来介绍 Tomcat的总体设计。从如何设计一个应用服务器开始，逐步完善，直至最终推导出 Tomcat的整体架构。



#### 2.1.1 Server



从最基本的功能来讲，我们可以将服务器描述为这样一个应用

> 它**接收**其他计算机（客户端）发来的**请求数据**并进行**解析**，**完成相关业务处理**，然后把处理**结果**作为响应**返回**给请求计算机（客户端）。

![](https://i.loli.net/2020/09/27/9nMmRBO3cbDEWzi.png)

- 我们通过`start()`方法启动服务器，**打开 Socket链接，监听服务器端口**，并负责在接收到客户端请求时进行处理并返回响应。

- 通过`stop()`方法来**停止服务器**并**释放网络资源**。



#### 2.1.2 Connector和Container



很快我们就会发现，将**请求监听**与**请求处理**放到 **一起** **扩展性很差**，比如当我们想适配多种网络协议，但是请求处理却相同的时候。



![](https://i.loli.net/2020/09/27/6pjmQqsnOVM7zC2.png)

- 一个Server可以**包含多个** Connector和 Container。
- 其中**Connector**负责开启 Socket并**监听客户端请求**，返回响应数据； 
- **Container**负责**具体的请求处理**。 
- **Connector**和 **Container**分别**拥有自己的`start()`和`stop()`方法**来**加载和释放**自己维护的资源。



但是，这个设计有个明显的**缺陷**。既然 Server可以包含多个 Connector和 Container，那么**如何知晓**来自**某个 Connector的请求**由**哪个 Container处理**呢？



![](https://i.loli.net/2020/09/27/PKVJ4Hgsz1yvGld.png)



- 一个Server包含多个 Service（它们互相独立，只是共享一个JVM以及系统类库）
- 一个**Service负责维护** **多个 Connector**和 **一个Container**，这样**来自 Connector的请求**只能由**它所属 Service维护的Container处理**。



#### 2.1.3 Container设计



**应用服务器**是用来部署并运行Web应用的，是一个**运行环境**，而不是一个独立的业务处理系统。因此，我们需要在 Engine容器中支持**管理多个Web应用**，当接收到 Connector的处理请求时， Engine容器能够找到一个合适的Web应用来处理。



![](https://i.loli.net/2020/09/27/B1KqUvlyL86xniS.png)



我们使用**Context**来表示一个**Web应用**，并且**一个 Engine**可以**包含多个 Context**



但是，设想我们有**一台主机**，它承担了**多个域名的服务**，`news.mycompany. com`、`article.mycompany.com`均由该主机处理，我们应如何实现呢？



既然我们要提供多个域名的服务，那么就可以将**每个域名**视为一个**虚拟主机**，在**每个虚拟主机下包含多个web应用**。因为对于客户端用户来说，他们并不了解服务端使用几台主机来为他们提供服务，只知道每个域名提供了哪些服务，因此，应用服务器将每个域名抽象为一个虚拟主机从概念上是合理的。



- 我们用**Host**表示**虚拟主机**的概念
- **一个Host**可以包含**多个Context**



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927142652.png)



>在 Tomcat的设计中， **Engine**既可以**包含Host**，又可以**包含 Context**，这是由具体的 Engine实现确定的，而且 Tomcat采用一种通用的概念解决此问题，我们在后续部分会详细讲解。Tomcat提供的**默认实现 Standard Engine只能包含 Host**



在一个**Web应用**中，可**包含多个Servlet实例**以**处理来自不同链接的请求**。因此，我们还需要一个组件概念来表示Servlet定义。在 Tomcat中， **Servlet**定义被称为**Wrapper**，



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927142920.png)



**容器**是指一类**处理接收自客户端的请求并且返回响应数据**的**组件**，前文提到的Engine、Host、Context、Wrapper均可属于**容器**。



我们使用 **Container**来表示容器

- Container可以添加并维护子容器，因此 Engine、Host、 Context、Wrapper均**继承**自 Container。
- 我们将它们之间的组合关系改为虚线，以表示它们之间是**弱依赖**的关系，即它们之间的关系是通过 Container的父子容器的概念体现的。
- 不过 **Service持有**的是 **Engine接口**（8.56版本之前为 Container接口，更加通用）。



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927143525.png)



> 既然 Tomcat的 **Container**可以表示**不同的概念级别**：Servlet引擎、虚拟主机、Web应用和Servlet，那么我们就**可以将不同级别的容器作为处理客户端请求的组件**，这**具体由**我们提供的**服务器的复杂度决定**。假使我们以嵌入式的方式启动 Tomcat，且运行极其简单的请求处理，不必支持多Web应用的场景，那么我们完全可以只在 Service中维护一个简化版的Engine。



此外， Tomcat的Container还有一个很重要的功能，就是**后台处理**。

- 在很多情况下，我们的Container需要执行一些**异步处理**，而且是**定期执行**，如每隔30秒执行一次， Tomcat对于Web应用文件变更的扫描就是通过该机制实现的。 
- Tomcat针对后台处理，在Container上**定义了`backgroundProcess()`方法**来**抽象后台处理**。



#### 2.1.4 Lifecycle



不难发现，所有**组件**均存在启动、停止等生命周期方法，**拥有生命周期管理的特性**。因此，我们**基于生命周期管理**进行一次**接口抽象**。

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927144159.png)



我们针对**所有拥有生命周期管理特性**的组件**抽象了一个 Lifecycle通用接口**，该接口定义了生命周期管理的核心方法。

- `init()`：初始化组件。
- `start()`：启动组件
- `stop()`：停止组件。
- `destroy()`：销毁组件。



同时，该接口**支持组件状态**以及**状态之间的转换**，支持**添加事件监听器**用于**监听组件的状态变化**。如此，我们可以采用一致的机制来初始化、启动、停止以及销毁各个组件。如**Tomcat核心组件**的默认实现均**继承自LifecycleMBeanBase抽象类**，该类不但**负责组件各个状态的转换**和**事件处理**。



Tomcat的Lifecycle接口状态图如下所示

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927144653.png)



#### 2.1.5 Pipeline和Value



在Tomcat中**每个Container组件**通过执行一个**职责链**来**完成具体的请求处理**。Tomcat**定义**了**Pipeline（管道）**和 **Valve（阀）**两个接口。前者用于**构造职责链**，后者代表**职责链上的每个处理器**。



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927145113.png)

- Pipeline中维护了一个**基础的Valve**，它始终**位于Pipeline的末端**（即最后执行），封装了具体的请求处理和输出响应的过程。

- 然后，通过**`addvalue()`方法**，我们可以**为Pipeline添加其他的Valve**。后添加的 Valve位于基础 Valve之前，并按照添加顺序执行。 

- Pipeline通过获得首个 Valve来启动整个链条的执行。

  

Tomcat容器组件的灵活之处在于，每个层级的容器（ Engine、Host、 Context、 Wrapper）均有对应的基础 Valve实现，同时维护了一个 Pipeline实例。也就是说，**我们可以在任何层级的容器上针对请求处理进行扩展**。



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927145520.png)





#### 2.1.6 Connector设计



接下来，我们再细化一下服务器设计中的另一个重要组件—**Connector**



要想与 Container配合实现一个完整的服务器功能， **Connector至少要完成如下几项功能。**

- **监听**服务器**端口**，**读取**来自客户端的**请求**。
- 将**请求数据**按照指定**协议**进行**解析**。
- 根据**请求地址** **匹配**正确的**容器**进行处理。
- 将**响应返回客户端**。



我们知道，Tomcat**支持多协议**，默认支持HTTP和AJP。同时，Tomcat还**支持多种I/O方式**，包括BIO（8.5版本之后移除）、NIO、APR。而且在Tomcat8之后新增了对NIO2和HTTP2协议的支持。因此，**对协议和I/O进行抽象和建模**是需要重点关注的。



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927151529.png)



- 在Tomcat中，**ProtocalHandler**表示一个**协议处理器**，针对**不同的协议和I/O方式**来提供**不同的实现**，如Http11NioProtocol表示NIO的HTTP协议处理器
- ProtocolHandler包含一个**Endpoint**用于**启动 Socket监听**，该接口**按照I/O方式**进行**分类实现**，如NIO2Endpoint表示非阻塞式Socket l/O。
  还包含一个**Processor**用于**按照指定协议读取数据**，并**将请求交由容器处理**，如Http11NIOProcessor表示在NIO的方式下HTTP请求的处理类。



在Connector启动时， Endpoint会启动线程来监听服务器端口，并在接收到请求后调用Processor进行数据读取。



当 Processor读取客户端请求后，需要**按照请求地址映射到具体的容器进行处理**，这个过程即为**请求映射**。

- Tomcat通过`Mapper`和 `MapperListener`两个类实现上述功能。
- 前者用于**维护容器映射信息**，同时按照映射规则（ Servlet规范定义）**查找容器**。
- 后者实现了 `Containerlistener`和 `LifecycleListener`，用于在容器组件状态发生变更时，**注册或者取消对应的容器映射信息**。
- Tomcat通过**适配器模式**（ Adapter）实现了 Connector与 Mapper、 Container的**解耦**。 Tomcat默认的 Connector实现（ Coyote）对应的适配器为`CoyoteAdapter`。





![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927151918.png)



#### 2.1.7 Executor



Tomcat提供了 **Executor接口**来表示一个可以**在组件间共享的线程池**（默认使用了JDK5提供的线程池技术），该接口**同样继承自Lifecycle**，可按照通用的组件进行管理。



线程池的共享范围如何确定？在 Tomcat中Executor由 Service维护，因此**同一个 Service中的组件可以共享一个线程池**。



在 Tomcat中， Endpoint会**启动一组线程**来**监听Socket端口**，当**接收到客户端请求**后，会**创建请求处理对象，并交由线程池处理**，由此支持并发处理客户端请求。



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927152234.png)

#### 2.1.8 Bootstrap和Catalina



Tomcat通过类**Catalina**提供了一个Shell程序，用于**解析**`server. xml`，并**创建各个组件**，同时，负责**启动、停止Tomcat应用服务器**。



Tomcat使用**Digester** **解析XML文件**，包括**server. xml**以及**web.xml**等，



最后， Tomcat提供了**Bootstrap**作为**应用服务器启动入口**。 **Bootstrap**负责**创建Catalina实例**，根据执行参数调用Catalina相关方法完成针对应用服务器的操作（启动、停止）。



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927152815.png)



Tomcat的启动方式可以作为非常好的示范来指导中间件产品设计。它实现了**启动入口**与**核心环境**的**解耦**，这样不仅**简化了启动**，而且**便于**我们更**灵活地组织中间件产品的结构**，尤其是**类加载器**的方案。



Tomcat也可以不通过Bootstrap和Catalina来启动服务器，Tomcat提供了一个同名类org.apache.catalina.startup.Tomcat，使用它我们可以将Tomcat服务器嵌入到我们的应用系统中并进行启动。



最后，通过一个表格来总结Tomcat服务器中的概念

| 组件名称  | 说明                                                         |
| --------- | ------------------------------------------------------------ |
| Server    | **Server**表示**整个Servlet容器**，因此 Tomcat运行环境中只有唯一一个Server实例 |
| Service   | **Service**表示**一个或者多个 Connector的集合**，这些 Connector**共享同一个 Container来处理请求**。在同一个 Tomcat实例内可以包含任意多个 Service实例，它们彼此独立 |
| Connector | 即**Tomcat链接器**，用于**监听并转化 Socket请求**，同时将读取的 Socket请求交由 Container处理，支持**不同协议**以及**不同的I/O方式** |
| Container | 表示**能够执行客户端请求并返回响应的一类对象**。在Tomcat中存在不同级别的容器：Engine、Host、 Context、 Wrapper |
| Engine    | Engine表示**整个Servlet引擎**。在 Tomcat中， Engine为最高层级的容器对象。尽管 Engine不是直接处理请求的容器，却是获取目标容器的入口 |
| Host      | Host作为一类容器，表示**Servlet引擎（即 Engine）中的虚拟机**，与一个服务器的网络名有关如域名等。客户端可以使用这个网络名连接服务器，这个名称必须要在DNS服务器上注册 |
| Context   | **Context作为一类容器**，用于表示 ServletContext，在 Servlet规范中，一个 ServletContext即**表示一个独立的Web应用** |
| Wrapper   | wrapper作为一类容器，用于表示**Web应用**中定义的**Servlet**  |
| Executor  | Executor表示Tomcat**组件间**可以**共享**的**线程池**         |



### 2.2 Tomcat启动



Tomcat的启动如下图

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927154104.png)



- 从图中我们可以看出， Tomcat的**启动过程**非常**标准化**，统一按照生命周期管理接口 Lifecycle的定义进行启动。
- 首先，调用`init()`方法进行组件的**逐级初始化**，然后再调用`start()`方法进行**启动**。当然，每次调用均伴随着生命周期状态变更事件的触发。
- **每一级组件**除完成自身的处理外，还要**负责调用子组件相应的生命周期管理方法**
- **组件与组件**之间是**松耦合的设计**，因此我们很容易通过配置进行修改和替换







### 2.3 请求处理



从本质上讲，应用服务器的请求处理开始于监听的 Socket端口接收到数据，结束于将服务器处理结果写入 Socket输出流。

- 在这个处理过程中，**应用服务器**需要**将请求**按照既定协议进行**读取**，并**封装**为**与具体通信方案无关**的**请求对象**。
- 然后根据**请求映射规则** **定位**到**具体的处理单元**。
- 当Servlet或者控制器的业务处理结束后，**处理结果**将被写入一个**与通信方案无关的响应对象**。
- 最后，该响应对象将**按照既定协议写入输出流**。



![](https://healthlung.oss-cn-beijing.aliyuncs.com/21312.jpg)



### 2.4 类加载器



本节将主要介绍 Tomcat的类加载机制，包括 Tomcat类加载器层级设计以及Web应用的类加载过程。类加载是一切Java应用运行的基础，了解一款应用的类加载机制会便于我们掌握它的运行边界，也有助于其运行时异常的快速定位。



#### 2.4.1 J2SE标准类加载器



JVM默认提供了三个类加载器，它们以一种**父子树**的方式**创建**，同时**使用委派模式**确保应用程序可通过自身的类加载器加载所有可见的Java类。结构如图2-19所示。



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200927155015.png)

- Bootstrap：用于加载**JVM提供**的**基础运行类**
- Extension：Java提供的一个标准的**扩展**机制用于加载**除核心类库外的Jar包**
- System：System类加载器通常用于**加载应用程序Jar包**及其**启动入口类**（ Tomcat的 Bootstrap类即由 System类加载器加载）



Java默认的**类加载机制**是**委派模式**，委派的过程如下：

1. 从**缓存中加载**。
2. 如果缓存中没有，则从**父类加载器**中加载。
3. 如果父类加载器没有，则从**当前类加载器**加载。
4. 如果没有，则**抛出异常**。



**应用程序**在**不自己构造类加载器**的情况下，**使用 System作为默认的类加载器**。如果应用程序**自己构造类加载器**，基本也**以 System作为父类加载器**。



#### 2.4.2 Tomcat加载器



Tomcat加载器要**考虑**如下几点**架构要素**

- **隔离性**：Web应用类库相互隔离，**避免依赖库**或者应用包**相互影响**。
- **灵活性**：既然Web应用之间的类加载器相互独立，那么我们就能**只针对一个Web应用进行重新部署**，此时该Web应用的类加载器将会重新创建，而且不会影响其他web应用。
- **性能**：由于每个Web应用都有一个类加载器，因此**web应用在加载类时，不会搜索其他Web应用包含的Jar包**，性能自然高于应用服务器只有一个类加载器的情况。



下面看一下Tomcat的类加载方案

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20200928122827.png)

 Tomcat提供了3个基础的类加载器和web应用类加载器，而且这3个类加载器指向的路径和包列表均可以由 catalina.properties配置

- Common：以 System为父类加载器，是位于Tomcat服务器顶层的公用类加载器。Common类加载器负责**加载Tomcat应用服务器内部和Web应用均可见的类**，例如 **Servlet规范相关包**和一些**通用的工具包**。
- Catalina：以 Common为父加载器，是用于加载 **Tomcat应用服务器**的类加载器，Catalina类加载器负责加载只有 Tomcat应用服务器内部可见的类，**这些类对Web应用不可见**，如 Tomcat的具体实现类
- Shared：以 Common为父加载器，是**所有web应用**的**父加载器**，Shared类加载器负责加载web应用共享的类，这些类 Tomcat服务器不会依赖。
- Web应用：以 Shared为父加载器，该类加载器**只对当前web应用可见**，对其他Web应用均不可见。



接下来，我们从架构层面讨论一下 Tomcat的类加载器方案。下面几点是对上述**架构分析的补充**：

- **共享**：
  - Tomcat通过 Common类加载器实现了Jar包在应用服务器以及Web应用之间共享，
  - 通过 Shared类加载器实现了Jar包在Web应用之间的共享，
  - 通过 Catalina类加载器加载服务器依赖的类。
  - 这样最大程度上实现了Jar包的共享，而且又确保了不会引入过多无用的包
- **隔离性**。这里的隔离性区别于前者，指**服务器与web应用的隔离**。理论上，除去 Servlet规范定义的接口外，我们的web应用不应依赖服务器的任何实现类，这样才有助于web应用的可移植性。正因如此， Tomcat支持通过 Catalina类加载器加载服务器依赖的包（尽管 Tomcat默认并没有这么做），以便应用服务器与Web应用更好地隔离。



#### 2.4.3 Web应用类加载器



Tomcat提供的web应用类加载器与默认的委派模式稍有不同。当进行类加载时，除JM基础类库外，它会首先尝试通过当前类加载器加载，然后才进行委派。

- 从缓存中加载。
- 如果没有，则从JVM的 Bootstrap类加载器加载。
- 如果没有，则从当前类加载器加载
- 如果没有，则从父类加载器加载，由于父类加载器采用默认的委派模式，所以加载顺序为 System、 Common、 Shared



Tomcat提供了 delegate属性用于控制是否启用Java委派模式，默认为 false（不启用）。



Tomcat通过该机制实现为**Web应用中的Jar包** **覆盖** **服务器提供包**的目的。如上所述，Java核心类库、 Servlet规范相关类库是无法覆盖的，此外Java默认提供的诸如XML工具包，由于位于JVM的 Bootstrap类加载器也无法覆盖，只能通过 endorsed的方式实现。