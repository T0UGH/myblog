---
title: '[Flink][8][读写外部系统]'
date: 2020-11-18 21:07:04
tags:
    - 大数据
    - Flink
    - 流处理
categories:
    - Flink
---
## 第8章 读写外部系统



数据可以**存储**在许多**不同的系统**中，比如**文件系统**、**对象存储**、**关系数据库系统**、**键值存储**、**搜索索引(search indexes)**、**事件日志**、**消息队列**等等。每一类系统都是为特定的访问模式而设计的，有各自擅长的领域。因此，当今的**数据基础设施**常常**由许多不同种类的存储系统组成**。



**数据处理系统**(如Apache  Flink)**通常不包含自己的存储层**，而是**依赖于外部存储系统**来摄取和持久存储数据。因此，对于像Flink这样的数据处理系统来说，**提供**一套齐全的从外部系统读取数据和向外部系统写入数据的**连接器库**以及**提供一套能够自定义连接器的API**是很重要的。



### 8.1 应用的一致性保障

 

要**保障应用的一致性**除了依赖于**Flink的检查点机制**之外，还需要**数据源和数据汇连接器**提供一些其他的特性与检查点机制配合。换句话说，应用的一致性保障依赖于三方面

- **检查点机制**
- 数据源连接器提供的**读取位置重置**
- 数据汇连接器提供的**幂等性写**或者**事务性写**



#### 8.1.1 数据源连接器提供的一致性保障



为了为应用程序提供**精确一次**的状态一致性保障，应用的每个**数据源连接器**都需要能够将其**读取位置重置**。

- 当**生成检查点**时，**数据源算子**将**存储其当前读取位置标识符**并在**需要恢复时利用远程存储来恢复这些位置标识符**。
- 支持**读取位置重置**的例子有:
  - **基于文件的数据源**（会存储文件字节流的读取偏置）
  - **Kafka数据源**（会存储消费主题分区的读取偏置）
- 如果**数据源连接器** **无法存储和重置**读取位置，那么此应用只能提供**最多一次保证**。



**读取位置重置**再加上**检查点机制**，就可以提供**至少一次的一致性保障**，如果还需要精确一次的一致性保障，那我们就需要数据汇连接器再提供一些其他的特性。



#### 8.1.2 数据汇连接器提供的一致性保障



##### 8.1.2.1 幂等性写



**幂等性**是指：对于**同一个系统**，在**同样条件**下，**一次请求**和**重复多次请求** **对资源的影响**是**一致**的，就称该操作为幂等的。



> 生活中的幂等操作
>
> 1. 博客系统同一个用户对同一个文章点赞，即使这人单身30年手速疯狂按点赞，那么实际上也只能给这个文章 +1 赞
> 2. 在微信支付的时候，一笔订单应当只能扣一次钱，那么无论是网络问题或者bug等而重新付款，都只应该扣一次钱



在Flink中我们主要在**数据汇连接器**上应用**幂等性的概念**来**提供端到端精确一次的一致性保障**。



> 举个简单的例子，假设我们要统计多个传感器的最高温度并且使用一个键值存储作为数据汇写入的外部系统。那么我们只要在数据汇算子中对**键值存储系统**进行**upsert操作**(看看key是否存在，存在就update，不存在就insert)就可以保证**幂等性**。因为对于一个传感器来说，多次插入数据，最终都只会产生一个k-v对。
>
> 
>
> 再具体点：
>
> 1. 经过计算，数据汇发出一条`("sensor_1", 100.0)`，然后这条记录被写入了键值存储系统。
> 2. 但是写入之后，发生了故障，系统恢复故障并重置到上一个检查点，经过重放，数据汇又发出了一条`("sensor_1", 100.0)`，然后这条记录也被写入了键值存储系统。
> 3. 但是幸运的是，因为**upsert**操作是幂等的，插入两条`("sensor_1", 100.0)`与插入一条`("sensor_1", 100.0)`对键值存储系统来说，影响是一样的。
> 4. 借此，我们成功保证了精确一次的一致性



##### 8.1.2.2 事务性写



第二种完全实现**端到端精确一次**的一致性方法是**事务性写**。这里的基本思想是只将那些位于上一个成功检查点之前的事件发送给外部系统。



相比于幂等性写，事务性写**延迟更高**，但是**不会重复发送事件**给外部系统，因此不会出现恢复过程中的不一致。



Flink中有两种事务性数据汇连接器

- write-ahead-log (WAL)数据汇
- 两阶段提交(2PC)数据汇



###### 8.1.2.2.1 WAL

- WAL数据汇将所有**事件** **先写入到算子状态**，并在**接收到检查点完成的通知**后再将它们**发送**给外部系统。
- WAL**通用性**很好，可以应对任何类型的外部系统（因为状态是缓存在算子状态中的）
- 但是，它不能提供100%的一致性保证，并且增加了应用的状态大小。



###### 8.1.2.2.2 2PC

- 2PC数据汇**需要外部系统提供对事务的支持**。
- 对于**每个检查点**，2PC数据汇**启动外部系统的一个事务**，并将所有接收到的记录写入到该事务。
- 当它**收到检查点完成的通知**时，它**提交事务**。
- 2PC协议依赖于Flink现有的检查点机制。
  - **检查点分隔符**可以作为**启动新事务**的指令
  - 所有**算子任务**关于其**单个检查点成功**向JobManager**发通知**可以看作是它们的**提交投票**
  - 而来自**JobManager**的**检查点成功**的消息是**提交事务**的指令。



下表展示了不同类型的数据源和数据汇连接器的搭配，能提供的一致性保障

| \                | 不可重置数据源 | 可重置数据源                                                 |
| ---------------- | -------------- | ------------------------------------------------------------ |
| **任意数据汇**   | 至多一次       | 至少一次                                                     |
| **幂等性数据汇** | 至多一次       | 精确一次(故障恢复过程中，会出现临时的不一致，故障恢复后，就恢复精确一次) |
| **WAL数据汇**    | 至多一次       | 至少一次(无法100%提供精确一次保障)                           |
| **2PC数据汇**    | 至多一次       | 精确一次                                                     |



### 8.2 内置连接器



Apache Flink提供了各种内置连接器来从各种外部系统读取数据和向各种外部系统写入数据。

- 像Kafka这样的消息队列和事件日志是常见的**外部源**
- 而消息队列、文件系统、键值系统、数据库系统是常见的**外部汇**



为了在应用中使用这些连接器，你需要将相应的依赖项添加到项目的构建文件中。



#### 8.2.1 Apache Kafka 数据源连接器



首先介绍**Kafka**

- Apache Kafka是一个分布式流处理平台。
- 它的核心是一个**分布式的发布-订阅消息系统**，该系统被广泛用于摄入(ingest)和分发(distribute)事件流。
- Kafka将**事件流**组织为所谓的**主题**。
  - 主题是一种事件日志(event log)，它保证事件顺序。
  - 我们可以将**主题划分为多个分区**，这些分区分布在一个集群中。
  - **有序保证仅限于单个分区**—kafka在从不同分区读取时不提供有序保证。Kafka在**分区中读取位置**称为**偏移量**(offset)。



Flink Kafka连接器可以**并行读取**事件流。

- **每个**数据源**算子任务**可以**从一个或多个分区读取**。
- **任务** **跟踪**每个分区的当前读取**偏移**，并**将偏置记录到其检查点数据**中。
- 从故障中恢复时，偏移量被取出，任务继续从检查点偏移量读取事件流。



图8-1显示了为数据源算子任务分配分区的情况

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201118142434.png)



下面看看如何创建一个Kafka数据源连接器

```scala
val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092")
properties.setProperty("group.id", "test")
val stream: DataStream[String] = env.addSource(
  new FlinkKafkaConsumer[String](
    "topic", // 主题
    new SimpleStringSchema(), // 反序列化
    properties)) // 配置参数
```



#### 8.2.2 Apache Kafka 数据汇连接器



下面看看如何创建一个Kafka数据汇连接器

```scala
val stream: DataStream[String] = ...
val myProducer = new FlinkKafkaProducer[String](
  "localhost:9092",         // broker服务器列表 
  "topic",                  // 当前主题
  new SimpleStringSchema)   // 序列化
stream.addSink(myProducer) // 设置为数据汇
```



##### 8.2.2.1 Kafka数据汇的至少一次保障



Kafka数据汇在**以下条件都满足**的情况下提供**至少一次的保证**:

- Flink的**检查点机制**是**开启**的
- 所有**数据源**都是**可重置**的
- 如果**写入未成功**，数据汇连接器要**抛出异常**（这样可以让应用失败然后恢复）
- **在提交检查点之前**，**数据汇连接器**要**等待** **Kafka** **确认**传输中的记录全部写入完毕。



##### 8.2.2.2 Kafka数据汇的精确一次保障



**Kafka支持事务性写**，因此Flink的Kafka数据汇也能够提供精确一次的一致性保障。但精确一次保障同样需要应用满足8.2.2.1中的几个条件。



FlinkKafkaProducer提供了一个带有Semantic参数的构造函数，该构造函数可以控制数据汇提供的一致性保证级别，该配置的选项如下:

- Semantic.NONE：**不提供**任何一致性保证——记录可能会丢失或多次写入
- Semantic.AT_LEAST_ONCE：**至少一次保证**，记录不会丢失，但可能会重复。这是默认设置。
- Semantic.EXACTLY_ONCE：**精确一次保证**



##### 8.2.2.3 CUSTOM PARTITIONING AND WRITING MESSAGE TIMESTAMPS



当向Kafka主题写入消息时，Flink Kafka**数据源算子任务**可以**选择**要写入主题的哪个**分区**。

- 你可以通过提供一个自定义的FlinkKafkaPartitioner来控制数据到主题分区的路由方式。
- 默认情况下，Flink将每个任务映射到一个Kafka分区。也就是说，由同一个任务发出的所有记录都写入到同一个分区。



通过调用数据汇上的`setWriteTimestampToKafka(true)`，可以**将记录的事件时间戳写入Kafka**。



#### 8.2.3 文件系统数据源连接器



文件系统通常用于以**高性价比**的方式**存储大量数据**。在大数据架构中，它们经常作为批处理应用程序的数据源和数据接收器。通过与高级文件格式(如Apache  Parquet或Apache ORC)相结合，**文件系统**可以有效地**为分析查询引擎**(如Apache Hive、Apache  Impala或Presto)**服务**。因此，文件系统通常用于**“连接”** **流处理应用**和**批处理应用**。



Apache Flink内置了一个文件数据源连接器，它**支持重置**，并可以将文件中的数据**作为数据流**输入到应用中。此外，它还支持多种类型的文件系统，比如：本地文件系统、HDFS、S3等等。



文件系统数据源的创建方法如下

````scala
val lineReader = new TextInputFormat(null) 
val lineStream: DataStream[String] = env.readFile[String](
  lineReader,                 // The FileInputFormat 文件输入格式，FileInputFormat的子类
  "hdfs:///path/to/my/data",  // The path to read 路径
  FileProcessingMode.PROCESS_CONTINUOUSLY,    // The processing mode 处理模式
  30000L) // The monitoring interval in ms 扫描的间隔时间
````



FileProcessingMode是目标路径的读取模式，有如下两种选择

- 在**PROCESS_ONCE**模式下，当**作业启动**并读取所有匹配的文件时，读取路径只被**扫描一次**。
- 在**PROCESS_CONTINUOUSLY**中，会**定期扫描路径**(在初始扫描之后)，并持续读取新的和修改的文件。



#### 8.2.4 文件系统数据汇连接器



将数据流写入文件是一种常见的需求。由于大多数应用只能在文件最终完成后读取文件，并且流应用运行时间很长，因此数据汇连接器通常会**将输出分块存储**到多个文件中。



当满足如下条件时，**文件系统数据汇连接器**可以为应用提供端到端的**精确一次保障**：

1. 应用开启**检查点机制**
2. 所有**数据源**都支持**重置**



下面看看如何创建一个文件系统数据汇连接器

```scala
val input: DataStream[String] = …
val sink: StreamingFileSink[String] = StreamingFileSink
  .forRowFormat(
    new Path("/base/path"),  //基础路径
    new SimpleStringEncoder[String]("UTF-8")) //编码器
  .build()
input.addSink(sink)
```



数据汇的文件分块机制分为三级：

- **数据流被划分为多个桶**
  - 每条记录都会被分配到一个桶中
  - 每个**桶**都**对应**基础路径下的一个**子路径**
  - 如果没有显式指定，Flink**根据事件的处理时间**将记录分配给**每小时一个**的桶中。
- **每个桶中包含多个part文件**
  - 每个bucket路径下包含多个part文件
  - 这些part文件由数据汇算子的**多个任务并发地写入**。
  - 此外，每个并行任务也将其输出分割成多个part文件。
  - 这些分块文件路径：`[base-path]/[bucket-path]/part-[task-idx]-[id]`
  - 例如，基础路径为`/johndoe/demo/`，分块文件前缀为`part`，则路径`/johndoe/demo/2018-07-22-17/part-4-8`指向2018年7月22日下午5点这个桶内(`/2018-07-22-17/`)，由第4个接收任务的写入的第8个文件(`/part-4-8`)。



StreamingFileSink提供了两种文件编码模式：**行编码**和**批量编码**。

- 在行编码模式中，每个记录都被单独编码然后写入到一个part文件中。我们调用`StreamingFileSink.forRowFormat()`方法就是使用行编码模式
- 在批量编码模式中，多条记录被一起编码然后写入到一个part文件中。我们调用`StreamingFileSink.forBulkFormat()`方法就是使用批量编码模式



#### 8.2.5 Apache Cassandra 数据汇连接器



Apache Cassandra是一种**可伸缩**的、**高可用**的**列式存储**数据库系统。

- Cassandra将数据集建模为由多个类型的列所组成的行表(Cassandra models datasets as tables of rows that consist of multiple typed columns.)。
- 可以将一个或多个列定义为(复合)主键。每一行都可以通过其主键唯一地标识。
- Cassandra提供了Cassandra查询语言(CQL)，这是一种类似sql的语言，用于读写记录以及创建、修改和删除数据库对象。



Flink的Cassandra连接器是可以**提供精确一次保障**的

- Cassandra的数据模型**基于主键**(K-V式的)，所有写到Cassandra的操作都使用**upsert语义**(有就update，没有就insert)。因此，基于Cassandra的数据汇写入操作是**幂等**的

- 与此同时，为了避免故障恢复期间的短暂不一致，Cassandra连接器可以通过配置**开启WAL机制**。



下面是例子



首先定义一个Cassandra表结构

````sql
// 创建键空间
CREATE KEYSPACE IF NOT EXISTS example
  WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'};
// 创建表
CREATE TABLE IF NOT EXISTS example.sensors (
  sensorId VARCHAR,
  temperature FLOAT,
  PRIMARY KEY(sensorId)
);
````



下面先展示如何将tuple、case class等类型写入Cassandra

```scala
val readings: DataStream[(String, Float)] = ???
// 新建数据汇构造器
val sinkBuilder: CassandraSinkBuilder[(String, Float)] = CassandraSink.addSink(readings)
// 构造
sinkBuilder
  .setHost("localhost")
  //CQL insert语句
  .setQuery("INSERT INTO example.sensors(sensorId, temperature) VALUES (?, ?);") 
  .build()
```

- 写入tuple、case class时需要指定CQL  INSERT查询。
- 数据汇将查询注册为PreparedStatement，并将tuple或case class的字段转换为PreparedStatement中的参数。
- **字段** **根据其位置**映射到参数；第一个值被转换为第一个参数。



下面再展示如何将POJO写入Cassandra

```scala
val readings: DataStream[SensorReading] = ???
CassandraSink.addSink(readings)
  .setHost("localhost")
  .build()
```

```scala
@Table(keyspace = "example", name = "sensors")
class SensorReadings(
  @Column(name = "sensorId") var id: String,
  @Column(name = "temperature") var temp: Float) {
  def this() = {
      this("", 0.0)
 }
  def setId(id: String): Unit = this.id = id
  def getId: String = id
  def setTemp(temp: Float): Unit = this.temp = temp
  def getTemp: Float = temp
}
```

- 需要通过**注解**来**映射**POJO字段到Cassandra列



### 8.3 实现自定义数据源函数



DataStream API提供了**两个接口**来**自定义数据源连接器**，这两个接口都有相应的RichFunction抽象类:

- SourceFunction用于**非并行**的数据源连接器
- ParallelSourceFunction用于支持**同时运行多个任务**的数据源连接器



这两个接口中的方法一样，如下所示

| 签名                             | 描述                                                         |
| -------------------------------- | ------------------------------------------------------------ |
| `void run(SourceContext<T> ctx)` | run()方法负责执行实际的事件读入工作。Flink会**开一个专门的线程**，然后在这个线程中调用run()方法一次。 |
| `void cancel()`                  | 负责**终止事件读入**                                         |



下面举一个简单的例子，它创建一个简单的从0数到Long.maxValue的自定义数据源函数

```scala
class CountSource extends SourceFunction[Long] {

  var isRunning: Boolean = true

  // 实际的数据读入工作   
  override def run(ctx: SourceFunction.SourceContext[Long]): Unit = {

    var cnt: Long = -1
    while (isRunning && cnt < Long.MaxValue) {
      // 不断的++
      cnt += 1
      ctx.collect(cnt)
    }

  }

  // 终止数据读入  
  override def cancel(): Unit = isRunning = false
}
```



#### 8.3.1 可重置的数据源函数



Flink只能在数据源连接器支持重放输入数据时才能提供各种一致性保证。

- 如果**外部源系统**提供了**读取偏移量**和**重置偏移量**的API，则**数据源算子可以重放输入数据**。
- 比如：**文件系统**。它提供文件流偏移量，它还提供将文件流移动到特定位置的seek方法，
- 再比如：**Apache  Kafka**，它为主题的每个分区提供偏移量，并可以设置分区的读取位置。
- 一个**反例是web socket**，它从网络套接字读取数据，并且立即丢弃交付的数据，因此它不支持重放



**支持重放的数据源算子**要与**检查点机制** **配合**

- 在**生成检查点**时，数据源算子要**持久化**所有当前**读取偏移量**。
- **故障恢复**时，数据源算子要从最新的检查点**恢复偏移量**到本地。



如果数据源算子要**支持重放**，则需要实现CheckpointedFunction接口，具体例子如下

```scala
// 可重置的SourceFunction
class ReplayableCountSource extends SourceFunction[Long] with CheckpointedFunction {

  var isRunning: Boolean = true
  var cnt: Long = _
  // 用来保存offset的算子列表状态  
  var offsetState: ListState[Long] = _
  // 实际的数据读入工作  
  override def run(ctx: SourceFunction.SourceContext[Long]): Unit = {

    while (isRunning && cnt < Long.MaxValue) {
      //加锁，但是讲道理我感觉只在这里加锁没什么用，除非框架进行了特殊的处理
      ctx.getCheckpointLock.synchronized { 
        // increment cnt
        cnt += 1
        ctx.collect(cnt)
      }
    }
  }

  override def cancel(): Unit = isRunning = false

  // 生成检查点时，这个方法被调用来同步状态  
  override def snapshotState(snapshotCtx: FunctionSnapshotContext): Unit = {
    // remove previous cnt
    offsetState.clear()
    // add current cnt
    offsetState.add(cnt)
  }

  // 初始化和故障恢复时会调用这个方法  
  override def initializeState(initCtx: FunctionInitializationContext): Unit = {
    // obtain operator list state to store the current cnt
    val desc = new ListStateDescriptor[Long]("offset", classOf[Long])
    offsetState = initCtx.getOperatorStateStore.getListState(desc)
	// 从检查点中恢复当前的进度
    // initialize cnt variable from the checkpoint
    val it = offsetState.get()
    cnt = if (null == it || !it.iterator().hasNext) {
      -1L
    } else {
      it.iterator().next()
    }
  }
}
```



#### 8.3.2 数据源函数、时间戳及水位线



DataStream API提供了**两种方式**来**分配时间戳**和**生成水位线**。

- 时间戳和水位线可以由专用的`TimestampAssigner`分配和生成
- 时间戳和水位线也可以由`SourceFunction`分配和生成。



SourceFunction的`SourceContext`对象提供了分配水位线和时间戳的方法

```java
@PublicEvolving
void collectWithTimestamp(T element, long timestamp);

@PublicEvolving
void emitWatermark(Watermark mark);
```



注意：当**单个数据源算子任务** **并行处理** **多个partition**时，使用`SourceFunction`来生成水位线比使用单独的`TimestampAssigner`更好，具体原因略、



### 8.4 实现自定义数据汇函数



DataStream API提供了SinkFunction接口来自定义数据汇函数。SinkFunction接口中只有一个方法:

```scala
void invoke(IN value, Context, ctx)
```



下面示例显示了一个简单的SinkFunction，它将传感器读数写入套接字。

```scala
// write the sensor readings to a socket
readings.addSink(new SimpleSocketSink("localhost", 9191))
// set parallelism to 1 because only one thread can write to a socket
.setParallelism(1)

//......

/**
  * Writes a stream of [[SensorReading]] to a socket.
  */
class SimpleSocketSink(val host: String, val port: Int)
    extends RichSinkFunction[SensorReading] {

  var socket: Socket = _
  var writer: PrintStream = _

  //设置socket连接和writer      
  override def open(config: Configuration): Unit = {
    // open socket and writer
    socket = new Socket(InetAddress.getByName(host), port)
    writer = new PrintStream(socket.getOutputStream)
  }

  //用writer写入到来的事件  
  override def invoke(
      value: SensorReading,
      ctx: SinkFunction.Context[_]): Unit = {
    // write sensor reading to socket
    writer.println(value.toString)
    writer.flush()
  }

  //关闭
  override def close(): Unit = {
    // close writer and socket
    writer.close()
    socket.close()
  }
}
```



为了实现**端到端精确一次保障**，数据汇连接器需要**幂等**或支持**事务**，下面我们看看这两种



#### 8.4.1 幂等性数据汇连接器



如果外部汇系统满足如下两个条件，SinkFunction接口就足以实现幂等性数据汇连接器

- 结果数据具有**固定的键**，可以**在该键上执行幂等更新**。
  - 对于计算每个传感器和每分钟的平均温度的应用程序，固定的键可以是传感器的ID和每分钟的时间戳。
- **外部系统支持按键更新**，例如关系数据库系统可以按照主键更新(update ... where sensor_id = xxx)或键值存储可以按照key更新



下面的例子演示了如何实现和使用一个幂等的SinkFunction，该函数将事件写入JDBC数据库。



```scala
class DerbyUpsertSink extends RichSinkFunction[SensorReading] {

  var conn: Connection = _
  var insertStmt: PreparedStatement = _
  var updateStmt: PreparedStatement = _

  // 初始化: 准备好insertStmt和updateStmt  
  override def open(parameters: Configuration): Unit = {
    // connect to embedded in-memory Derby
    val props = new Properties()
    conn = DriverManager.getConnection("jdbc:derby:memory:flinkExample", props)
    // prepare insert and update statements
    insertStmt = conn.prepareStatement(
      "INSERT INTO Temperatures (sensor, temp) VALUES (?, ?)")
    updateStmt = conn.prepareStatement(
      "UPDATE Temperatures SET temp = ? WHERE sensor = ?")
  }

  // 实际的处理函数: 先尝试更新失败就插入 
  override def invoke(r: SensorReading, context: Context[_]): Unit = {
    // set parameters for update statement and execute it
    updateStmt.setDouble(1, r.temperature)
    updateStmt.setString(2, r.id)
    updateStmt.execute()
    // execute insert statement if update statement did not update any row
    if (updateStmt.getUpdateCount == 0) {
      // set parameters for insert statement
      insertStmt.setString(1, r.id)
      insertStmt.setDouble(2, r.temperature)
      // execute insert statement
      insertStmt.execute()
    }
  }

  // 关闭: 关闭stmt和conn  
  override def close(): Unit = {
    insertStmt.close()
    updateStmt.close()
    conn.close()
  }
}
```



#### 8.4.2 事务性数据汇连接器



为了简化事务性数据汇的实现，Flink的DataStream API提供了两个模板，可以通过继承这些模板来实现自定义数据汇算子。两个模板都实现了CheckpointListener接口来接收JobManager发出的检查点已完成的通知



- **GenericWriteAheadSink**模板暂存每个检查点周期内所有需要发出到外部系统的记录，并将它们存储在数据汇算子任务的算子状态中。当任务接收到检查点完成通知时，它将已完成的那个检查点周期内的记录写入外部系统。
- **TwoPhaseCommitSinkFunction**模板**利用了外部汇系统的事务特性**。对于每个检查点，它启动一个新事务，并将这个检查点周期内的事件写入到这个事务中。



##### 8.4.2.1  GenericWriteAheadSink(WAL)



GenericWriteAheadSink的工作方式如下：

- 它将所有接收到的记录添加(append)到使用检查点来分段(segmented)的**write ahead log(**WAL)中。
- 每次数据汇算子接收到**检查点分隔符**时，它**将开启一个新的section**，并将以下所有记录追加到新section。
- **WAL作为算子状态**被存储，当生成检查点时，WAL被发送给远程存储。
- 由于**WAL可以在出现故障时恢复**，因此不会丢失任何记录。



当GenericWriteAheadSink收到关于已完成检查点的通知时，它会将WAL中对应这个检查点的section中所有记录发送给外部系统。



当所有记录都被成功发出后，GenericWriteAheadSink将在**内部提交**相应的**检查点**。检查点需要通过两个步骤提交。

- 首先，数据汇**永久存储**"这个检查点已提交了"这条**提交信息**。
- 其次，它**从WAL中删除**对应的section中全部**记录**。
- 注意：**提交信息不存储算子状态中**。相反，GenericWriteAheadSink依赖于一个称为**CheckpointCommitter的可插入组件**来**存储和查找提交信息**。



下面看看如何利用GenericWriteAheadSink来实现一个事务性数据汇连接器

````scala
avgTemp.transform(
        "WriteAheadSink",
        new StdOutWriteAheadSink)
      // enforce sequential writing
      .setParallelism(1)
/**
  * Write-ahead sink that prints to standard out and commits checkpoints to the local file system.
  */
// GenericWriteAheadSink抽象类需要传入三个参数
// 1 用于存储提交信息的CheckpointCommitter
// 2 TypeSerializer 用来序列化
// 3 任务id
class StdOutWriteAheadSink extends GenericWriteAheadSink[(String, Double)](
    // CheckpointCommitter that commits checkpoints to the local file system
    new FileCheckpointCommitter(System.getProperty("java.io.tmpdir")),
    // Serializer for records
    createTypeInformation[(String, Double)].createSerializer(new ExecutionConfig),
    // Random JobID used by the CheckpointCommitter
    UUID.randomUUID.toString) {

  // 并且我们需要自己实现sendValues方法  
  override def sendValues(
      readings: Iterable[(String, Double)],
      checkpointId: Long,
      timestamp: Long): Boolean = {

    for (r <- readings.asScala) {
      // write record to standard out 将输入写入到标准输出中(就举个简单例子)
      // 运行时可以看到stdout中没有重复记录也不少记录  
      println(r)
    }
    true
  }
}
````



如前所述，GenericWriteAheadSink不能提供100%的精确一次一致性保证。有两种失败情况会导致记录被多次发出:

- 程序在任务**运行sendValues()方法时发生故障**。
  - 这时，单个section中有的记录被写入了外部系统但是有的没有写入。
  - 然后由于检查点尚未提交，当恢复时，数据汇将再次写入这个section中的所有记录。
  - 从而导致记录多次发送。
- 所有记录都被正确写入，sendValues()方法返回true；但是，**程序在调用CheckpointerCommitter之前失败，或者CheckpointerCommitter未能提交检查点**。这样，在恢复期间，系统会误认为这个checkpoint未内部提交，并再次写入这个section中的所有记录。



##### 8.4.2.1  TwoPhaseCommitSinkFunction(2PC)



TwoPhaseCommitSinkFunction是这样实现2PC协议的。

- 在数据汇向外部汇系统发出它的第一个记录之前，它在外部汇系统上启动一个事务。
- 这之后收到的所有记录都写入到这个事务中。
- 当JobManager**启动一个检查点并在数据源中放入检查点分隔符时**，2PC协议的**投票阶段开始**。
- 当**一般算子**接收到检查点分隔符时，它向**远程存储**同步自己的状态，并在完成之后**向JobManager发送确认消息**。
- 当**数据汇算子任务**接收到检查点分隔符时，它向**远程存储**同步自己的状态，准备好要提交事务，并在完成之后**向JobManager发送确认消息**。
- JobManager收到的确认消息类似于2PC协议的提交投票。**数据汇任务此时还不能提交事务**，因为不能保证所有其他数据汇任务都到达了检查点。此时**数据汇任务会再开启一个新的事务**，将到来的记录写入到这个新事务中。
- 当JobManager**从所有任务上接收到确认消息**时，它将**检查点完成通知** **发送**给所有订阅通知的任务。此通知对应于2PC协议的commit命令。
- 当**数据汇任务收到通知**时，它**提交检查点对应的事务**。
- **当所有的数据汇任务都提交了它们的事务**时，2PC协议的迭代就**成功**了。



要实现2PC协议，还需要外部汇系统满足一些要求

- **外部汇系统**必须**提供事务支持**。
- 在**检查点间隔**期间，**事务**必须**打开**并接受**写**操作。
- **事务必须直到收到检查点完成通知才能提交**。在恢复周期的情况下，这可能需要一些时间。如果接收系统关闭事务(例如因超时关闭了事务)，未提交的数据将丢失。
- 外部汇系统必须能够在进程**失败**后**恢复**事务。
- **提交事务必须是幂等操作**——外部汇系统应该能够注意到事务已经提交，或者重复提交必须没有效果。



下面实例将2PC作用在一个文件系统上

```scala
class TransactionalFileSink(val targetPath: String, val tempPath: String)
	// IN => (String, Double) 输入记录的类型 
	// TXT => String 事务标识符的类型
	// CONTEXT => Void 上下文类型，void代表不需要上下文
    extends TwoPhaseCommitSinkFunction[(String, Double), String, Void](
      // 序列化器  
      createTypeInformation[String].createSerializer(new ExecutionConfig),
      // 序列化器  
      createTypeInformation[Void].createSerializer(new ExecutionConfig)) {

  var transactionWriter: BufferedWriter = _

  /**
    * 当需要开启新事务的时候就新建一个临时文件
    */
  override def beginTransaction(): String = {

    // path of transaction file is constructed from current time and task index
    val timeNow = LocalDateTime.now(ZoneId.of("UTC"))
      .format(DateTimeFormatter.ISO_LOCAL_DATE_TIME)
    val taskIdx = this.getRuntimeContext.getIndexOfThisSubtask
    // 文件名 = 时间 + 任务id  
    val transactionFile = s"$timeNow-$taskIdx"

    // 创建文件和writer
    val tFilePath = Paths.get(s"$tempPath/$transactionFile")
    Files.createFile(tFilePath)
    this.transactionWriter = Files.newBufferedWriter(tFilePath)
    println(s"Creating Transaction File: $tFilePath")

    // 文件名被返回，作为事务的标识符
    transactionFile
  }

  /** Write record into the current transaction file. */
  /** 将记录写入到当前临时文件. */
  override def invoke(transaction: String, value: (String, Double), context: Context[_]): Unit = {
    transactionWriter.write(value.toString)
    transactionWriter.write('\n')
  }

  /** Flush and close the current transaction file. */
  override def preCommit(transaction: String): Unit = {
    transactionWriter.flush()
    transactionWriter.close()
  }

  /** Commit a transaction by moving the pre-committed transaction file
    * to the target directory.
    */
  /** 通过将临时文件移动到目标路径来提交事务    */ 
  override def commit(transaction: String): Unit = {
    val tFilePath = Paths.get(s"$tempPath/$transaction")
    // check if the file exists to ensure that the commit is idempotent.
    if (Files.exists(tFilePath)) {
      val cFilePath = Paths.get(s"$targetPath/$transaction")
      Files.move(tFilePath, cFilePath)
    }
  }

  /** Aborts a transaction by deleting the transaction file. */
  /** 通过将临时文件删除来关闭事务    */ 
  override def abort(transaction: String): Unit = {
    val tFilePath = Paths.get(s"$tempPath/$transaction")
    if (Files.exists(tFilePath)) {
      Files.delete(tFilePath)
    }
  }
}
```



### 8.5 异步访问外部系统



除了接收(ingest)或发出(emit)数据流之外，**通过在远程数据库中查找信息来丰富数据流**是另一个需要与外部存储系统交互的常见场景。比如：雅虎的广告分析系统，它需要使用存储在键值存储中的对应广告的详细信息来丰富广告点击流。



对于这类场景，**最直接的方法是实现一个MapFunction**，它为每个记录查询外部系统，等待查询返回结果，丰富记录，并发出结果。虽然这种方法很容易实现，但存在一个主要问题:**对外部系统的每个请求都会增加显著的延迟**(一个请求/响应涉及两个网络消息)，而MapFunction将大部分时间用于等待查询结果。



Flink提供了**AsyncFunction**来**减轻远程I/O调用的延迟**。AsyncFunction**并发地发送**多个**查询**并**异步地处理**它们的**结果**。



下面来看看`AsyncFunction`的源代码

```scala
trait AsyncFunction[IN, OUT] extends Function {
  // input: 输入记录
  // ResultFuture[OUT]: 用于返回函数结果的异步Future对象(也有可能返回一个异常)  
  def asyncInvoke(input: IN, resultFuture: ResultFuture[OUT]): Unit
}
```



下面展示如何使用`AsyncFunction`来并发查询关系型数据库(这里使用Derby数据库)

```scala
val readings: DataStream[SensorReading] = ???
val sensorLocations: DataStream[(String, String)] = 
AsyncDataStream
  .orderedWait(
    readings,
    new DerbyAsyncFunction,
    5, TimeUnit.SECONDS,    // timeout requests after 5 seconds
    100)                    // at most 100 concurrent requests
// .....
class DerbyAsyncFunction extends AsyncFunction[SensorReading, (String, String)] {

  // caching execution context used to handle the query threads
  private lazy val cachingPoolExecCtx =
    ExecutionContext.fromExecutor(Executors.newCachedThreadPool())
  // 用于将结果Future转发给回调
  private lazy val directExecCtx =
    ExecutionContext.fromExecutor(
      org.apache.flink.runtime.concurrent.Executors.directExecutor())

  /** Executes JDBC query in a thread and handles the resulting Future
    * with an asynchronous callback. */
  override def asyncInvoke(
      reading: SensorReading,
      resultFuture: ResultFuture[(String, String)]): Unit = {

    val sensor = reading.id

    // 创建一个Future，从数据库中获取room信息
    val room: Future[String] = Future {
      // Creating a new connection and statement for each record.
      // Note: This is NOT best practice!
      // Connections and prepared statements should be cached.
      val conn = DriverManager
        .getConnection("jdbc:derby:memory:flinkExample", new Properties())
      val query = conn.createStatement()

      // submit query and wait for result. this is a synchronous call.
      val result = query.executeQuery(
        s"SELECT room FROM SensorLocations WHERE sensor = '$sensor'")

      // get room if there is one
      val room = if (result.next()) {
        result.getString(1)
      } else {
        "UNKNOWN ROOM"
      }

      // close resultset, statement, and connection
      result.close()
      query.close()
      conn.close()

      // sleep to simulate (very) slow requests
      Thread.sleep(2000L)

      // return room
      room
    }(cachingPoolExecCtx) // 第二个参数是一个线程池

    // 给room这个Future注册一个回调
    room.onComplete {
      case Success(r) => resultFuture.complete(Seq((sensor, r)))
      case Failure(e) => resultFuture.completeExceptionally(e)
    }(directExecCtx)
  }
}
```



- asyncInvoke()方法包装了阻塞式的JDBC查询
- 这些JDBC查询通过CachedThreadPool执行。
- Future[String]返回JDBC查询的结果。
- 最后，我们调用Future对象(也就是上面代码的room常量)onComplete()回调，并将查询结果传递给ResultFuture handler。
- 将查询结果传递给ResultFuture handler是由DirectExecutor处理的。



值得一提的是，**`asyncInvoke`方法本身**与其他算子函数没有区别，都是**串行调用**的。只不过它在内部使用了Future对象来发出并回收异步请求。

