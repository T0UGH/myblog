---
title: '[Flink][5][DataStreamAPI]'
date: 2020-11-8 19:09:54
tags:
    - 大数据
    - Flink
    - 流处理
categories:
    - Flink
---
## 第5章 DataStreamAPI



本章介绍了Flink的DataStream  API的**基础知识**。我们将展示**常用的Flink流应用程序**的结构和组件，讨论Flink的**类型系统**和支持的数据类型，并介绍**数据转换**(data transformation)和**分区转换**(partitioning transformation)。读完这一章，你将知道**如何实现一个具有基本功能的流处理应用程序**。



### 5.1 Hello,Flink!



首先举一个简单的例子作为开始



````scala
/** 定义一个case class作为传感器读取数据的数据类型*/
/** Case class to hold the SensorReading data. */
case class SensorReading(id: String, timestamp: Long, temperature: Double)

/** 放主函数的object*/
/** Object that defines the DataStream program in the main() method */
object AverageSensorReadings {
    
  /** main() defines and executes the DataStream program */
  def main(args: Array[String]) {

    // 设置流式执行环境  
    // set up the streaming execution environment
    val env = StreamExecutionEnvironment.getExecutionEnvironment
	
    // 在应用中使用事件时间  
    // use event time for the application
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    
    // 配置水位线
    // configure watermark interval
    env.getConfig.setAutoWatermarkInterval(1000L)

    // 从流式数据源中创建DataStream[SensorReading]对象  
    // ingest sensor stream
    val sensorData: DataStream[SensorReading] = env
      // 添加传感器Source
      // SensorSource generates random temperature readings
      .addSource(new SensorSource)
      // 设置时间戳和水位线
      // assign timestamps and watermarks which are required for event time
      .assignTimestampsAndWatermarks(new SensorTimeAssigner)

    val avgTemp: DataStream[SensorReading] = sensorData
      // 将温度从华氏温度转换为摄氏温度
      // convert Fahrenheit to Celsius using an inlined map function
      .map( r =>
      SensorReading(r.id, r.timestamp, (r.temperature - 32) * (5.0 / 9.0)) )
      // 根据传感器id来分组数据
      // organize stream by sensorId
      .keyBy(_.id)
      // 按照1秒的滚动窗口分组
      // group readings in 1 second windows
      .timeWindow(Time.seconds(1))
      // 使用用户自定义函数来计算平均温度
      // compute average temperature using a user-defined function
      .apply(new TemperatureAverager)

    // 打印到控制台  
    // print result stream to standard out
    avgTemp.print()

    // 开始执行应用  
    // execute application
    env.execute("Compute average sensor temperature")
  }
}
````





构建一个典型的Flink流式程序需要以下几步

1. 设置执行**环境**
2. 从**数据源**中读取一条或多条流
3. 通过一系列**流式转换**来实现**应用逻辑**
4. 选择性地将结果输出到一个或多个**数据汇**中
5. **执行**程序



#### 5.1.1 设置执行环境



Flink应用程序需要做的第一件事是**设置它的执行环境**。执行环境确定程序是**在本地机器上运行**还是**在集群上运行**。在DataStream API中，应用程序的执行环境由`StreamExecutionEnvironment`表示。

有两种设置执行环境的方式

1. 调用静态`getExecutionEnvironment()`方法来**检索执行环境**。此方法返回本地或远程环境，具体**取决于**调用该方法的**上下文**。如果通过连接到远程集群从提交客户端调用该方法，则返回远程执行环境。否则，它返回一个本地环境。

2. 也可以通过`createxxx`方法来显式设置执行环境，具体代码如下

   ````scala
   // 创建一个本地的流式执行环境
   val localEnv = StreamExecutionEnvironment.createLocalEnvironment()
   // 创建一个远程的流式执行环境
   val remoteEnv = StreamExecutionEnvironment.createRemoteEnvironment(
   	"host", 				// JobManager的主机名
       1234,					// JobManager的端口号
       "path/to/jarFile.jar"	// 需要传输到JobManager的JAR包
   )
   ````



**执行环境**还提供了很多**配置选项**，比如

1. 通过`env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)`指定当前应用使用**事件时间**语义
2. 设置**并行度**
3. 启动**容错**等



#### 5.1.2 读取输入流



`StreamExecutionEnvironment`提供了一系列**创建流式数据源**的**方法**，用来将数据流读取到应用中。这些数据流的**来源**可以是**消息队列**或者**文件**，也可以**动态生成**。



在实例中，读取代码如下

````scala
// 从流式数据源中创建DataStream[SensorReading]对象  
// ingest sensor stream
val sensorData: DataStream[SensorReading] = env
    // 添加传感器Source
    // SensorSource generates random temperature readings
    .addSource(new SensorSource)
    // 设置时间戳和水位线
    // assign timestamps and watermarks which are required for event time
    .assignTimestampsAndWatermarks(new SensorTimeAssigner)
````



#### 5.1.3 应用转换(Apply Transformation)



当获取到了`DataStream`，就可以对其应用转换(we can **apply a transformation** on it)。转换的类型有很多：

1. 有些转换可以生成新的`DataStream`，并且可能是不同类型的(eg. `DataStream[Int] => DataStream[String]`)
2. 有些转换不修改`DataStream`中的条目，而是通过**分区**或**分组**对其进行**重新组织**。
3. **应用程序的逻辑**是通过一系列**转换**定义的。



在实例中，转换代码如下

```scala
val avgTemp: DataStream[SensorReading] = sensorData
    // 将温度从华氏温度转换为摄氏温度
    // convert Fahrenheit to Celsius using an inlined map function
    .map( r =>
    SensorReading(r.id, r.timestamp, (r.temperature - 32) * (5.0 / 9.0)) )
    // 根据传感器id来分组数据
    // organize stream by sensorId
    .keyBy(_.id)
    // 按照1秒的滚动窗口分组
    // group readings in 1 second windows
    .timeWindow(Time.seconds(1))
    // 使用用户自定义函数来计算平均温度
    // compute average temperature using a user-defined function
    .apply(new TemperatureAverager)
```



#### 5.1.4 输出结果



流应用程序通常将其结果发送到一些**外部系统**(external system)，如Apache  Kafka、文件系统或数据库。Flink提供了一组**流式数据汇**，可用于**将数据写入不同的系统**。也可以实现自己的流式数据汇。还有**一些**应用程序**不发出结果**，而是通过Flink的可查询状态(queryable state)功能在**内部保存**结果。



在我们的示例中，会将`DataStream[SensorReading]`中的记录作为结果输出。每个记录包含传感器在5秒内的平均温度。通过调用print()将结果流写入标准输出:

```scala
avgTemp.print()
```



#### 5.1.5 执行



当应用定义完成后，可以通过调用`StreamExecutionEnvironment.execute()`来执行它:

```scala
env.execute("Compute average sensor temperature")
```



Flink程序都是通过**延迟计算**(lazily execute)的方式执行。

- 也就是说，那些创建数据源和转换操作的API调用不会立即触发任何实际的数据处理。
- 相反，这些API调用只是**在执行环境**中创建一个**执行计划**。该**计划包括**从环境创建的**流式数据源**以及应用于这些数据源之上的**一系列转换**。
- 只有在调用`execute()`时，系统**才会触发程序的执行**。



构建完成的计划会被转换为**JobGraph**并**提交给JobManager**执行。

- 根据**执行环境的类型**，系统可能需要将JobGraph发送到作为**本地**线程启动的JobManager，或将JobGraph发送到**远程**JobManager。
- 如果JobManager**远程**运行，除了JobGraph之外，我们**还需要**提供一个包含应用程序的所有类和所需依赖项的**JAR文件**。

### 5.2 转换操作



在本节中，我们将概述DataStream  API中的**基本转换**。

- 流式转换以一个或多个数据流作为输入，并将它们转换为一个或多个输出流。
- 编写一个DataStream API程序**本质上**可以归结为：通过**组合**不同的**转换**来**创建**一个**满足应用逻辑**的**Dataflow图**。



大多数流式转换都基于用户自定义的函数来完成。**这些函数封装了用户的逻辑**，指定了如何将输入流的元素转换为输出流的元素。**函数**可以**通过实现**某个特定转换的**接口类**来定义，例如下面的`MapFunction`

```scala
class MyMapFunction extends MapFunction[Int, Int] {
    override def map(value: Int): Int = value + 1
}
```





DataStream API为那些最常见的数据转换操作都提供了对应的转换抽象，我们将DataStreamAPI的**转换分为四类**

1. 作用于**单个事件**的**基本转换**
2. 针对**相同键值**事件的**KeyedStream转换**
3. 将多条数据流**合并**为一条或将一条数据流**拆分**成多条流的转换
4. 对流中的事件进行**重新组织**的**分发转换**



#### 5.2.1 基本转换



基本转换**单独处理**每个事件，这意味着**每个输出记录**都是**由单个输入记录生成**的。常见的基本转换函数有：简单的值转换、记录拆分或过滤等。



##### 5.2.1.1 Map



通过调用`DataStream.map()`方法可以指定**map转换**来**产生一个新的**`DataStream`。它将每个**输入事件**传递给用户自定义的映射器(user-defined mapper)，**映射器返回一个输出事件**，这个输出事件可能是不同类型的(eg, DataStream[Int] => DataStream[String])。图5-1显示了将每个正方形转换为圆形的map转换。

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201108153218.png)

MapFunction的两个类型参数分别是是输入事件的类型和输出事件的类型，`MapFunction`的`map()`方法将每个输入事件准确地转换为一个输出事件:

```scala
// T: 输入元素的类型
// O: 输出元素的类型
MapFunction[T,O] 
	> map(T): O
```



下面举一个简单的例子

````scala
val sensorIds: DataStream[String] = reading.map(new MyMapFunction)

class MyMapFunction extends MapFunction[SensorReading, String] {
    override def map(r: SensorReading): String = r.id
}
````

也可以用Lambda表达式进一步简化

````scala
val sensorIds: DataStream[String] = reading.map(r => r.id)
````



##### 5.2.1.2 Filter



**fliter转换**通过一个**返回值为`Boolean`类型的函数**来**决定事件的去留**：

- 如果返回值为true，那么它会保留输入事件并且将其转发到输出，
- 否则它会把事件丢弃。
- 通过调用**DataStream.filter()**方法可以指定过滤器转换，并生成与输入DataStream**相同类型**的输出DataStream。
- 图5-2显示了一个只保留白色方块的过滤操作。

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201108154544.png)



`FilterFunction`**类型参数**是**输入流的类型**，它的`filter()`方法**接收一个输入事件**并**返回一个布尔值**:

```scala
FilterFunction[T]
	> filter(T): Boolean
```



下面举个简单的例子

````scala
var filteredSensors = readings.filter(r => r.temperature >= 25)
````





##### 5.2.1.3 FlatMap



flatMap转换与map类似，但是它可以为每个输入事件生成**零个**、**一个**或**多个** **输出事件**。

图5-3显示了一个基于传入事件的颜色区分其输出的flatMap操作。

- 如果输入是白色方块，则不加改动直接输出。
- 将黑色方块复制，
- 将灰色方块丢弃掉。



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201108155131.png)



flatMap函数定义如下，可以通过向`Collector`对象传递数据的方式来返回零个、一个或多个事件作为结果

```scala
// T: 输入元素的类型
// O: 输出元素的类型
FlatMapFunction[T, O]
	// 返回值为Unit，也就是不返回
	// Collector[O]作为输出参数
	> flatMap(T, Collector[O]): Unit
```

flatMap函数还可以如下定义

````scala
FlatMapFunction[T, O]
	> flatMap(T):  TraversableOnce[O]
````



下面举一个简单的例子

```scala
val words = sensorData.flatMap(r => r.id.split(" "))
```



#### 5.2.2 基于KeyedStream的转换



KeyedStream抽象可以从逻辑上**将事件按照键值**分配到**多个独立的事件子流**中。



KeyedStream可以根据键来维护内部状态，所有具有相同键的事件可以访问相同的状态。



接下来先介绍**keyBy转换**，它可以将要一个`DataStream`转换为一个`KeyedStream`。然后介绍**滚动聚合**和**Reduce**，它们可以作用在`KeyedStream`上



##### 5.2.2.1 keyBy



**keyBy**转换通过**键**来**将DataStream转换为KeyedStream**。数据流中的事件会根据不同的键被分配到不同的分区(**partition**)，具有**相同键的所有事件**都由**下游算子**的**同一个任务**处理。



我们假设以输入事件的颜色作为键，图5-4将黑色事件分配给一个分区，将所有其他事件分配给另一个分区。

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201108161237.png)



keyBy可以用多种方式来设置如何分类，如下所示

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201108161457.png)



下面举一个keyBy的例子

````scala
val readings: DataStream[SensorReading] = ...
val keyed: KeyedStream[SensorReading, String] = readings.keyBy(r => r.id)
````



##### 5.2.2.2 滚动聚合



**滚动聚合** **应用于KeyedStream**上，它生成一个包含**聚合结果**（如求和、最小值和最大值）的DataStream。

- 滚动聚合操作符为**每个键**保存**一个聚合值**。
- 对于每个**输入事件**，算子**更新**相应的**聚合值**，并将**更新后的值**作为输出事件**发送给下游**。
- 滚动聚合操作需要接收一个用于指定**聚合目标字段**的参数，该参数指定在哪个字段上计算聚合。



DataStream API提供了以下滚动聚合方法:

| 名称      | 描述                                             |
| --------- | ------------------------------------------------ |
| `sum()`   | 滚动计算输入流在指定字段上的**和**               |
| `max()`   | 滚动计算输入流在指定字段上的**最大值**           |
| `min()`   | 滚动计算输入流在指定字段上的**最小值**           |
| `minBy()` | 滚动计算输入流中迄今为止最小值，返回该值所在事件 |
| `maxBy()` | 滚动计算输入流中起劲为止最大值，返回该值所在事件 |



注意：**不能**将多个**滚动聚合**方法**组合**使用，**每次只能计算一个**。



例子：对一个`Tuple3[Int, Int,  Int]`类型的数据在第一个字段上按照键值分组，然后滚动计算第二个字段的和

````scala
val inputStream: DataStream[(Int, Int, Int)] = env.fromElements(
    (1, 2, 2),
    (2, 3, 1),
    (2, 2, 4),
    (1, 5, 3))

val resultStream: DataStream[(Int, Int, Int)] = inputStream
	.keyBy(0)
	.sum(1)

"""output
(1, 2, 2)
(2, 3, 1)
(2, 5, 1)
(1, 7, 2)
第一个字段是分组，第二个字段是计算之后的和，第三个字段没有意义
"""
````



##### 5.2.2.3 Reduce



reduce转换是**滚动聚合**的**一般化**(generalization)。

- 它在KeyedStream上应用了一个ReduceFunction，该函数将每个**输入事件**与**当前的reduce结果**进行一次**组合**，并输出一个DataStream。
- reduce**不会改变DataStream的类型**，输出流的类型与输入流的类型相同。



ReduceFunction接口定义如下

```scala
// T: 元素类型
ReduceFunction[T]
	> reduce(T, T): T
```



下面举一个reduce转换的例子。在下面的例子中，数据流是会以语言类型作为键来进行分区，最终结果是针对每个语言产生一个不断更新的单词列表:

````scala
val inputStream: DataStream[(String, List[String])] = env.fromElements(
    ("en", List("tea")),
    ("fr", List("vin")),
    ("en", List("cake")))

val resultStream: DataStream[(String, List[String])] = inputstream
	.keyBy(0)
	.reduce((x, y) => (x._1, x._2 ::: y._2))

"""output
("en", List("tea"))
("fr", List("vin"))
("en", List("tea", "cake"))
"""
````



#### 5.2.3 多流转换



许多应用需要将多个输入流联合起来处理，还有一些应用需要将一条流分割成多条子流以应用不同的逻辑。下面，我们将讨论那些同时处理多个输入流或产生多个输出流的DataStream API转换。



##### 5.2.3.1 Union



`DataStream.union()`方法可以合并两个或多个**相同类型**的`DataStream`，并生成一个新的类型相同的`DataStream`。



图5-5显示了一个union操作，它将黑色和灰色事件合并到单个输出流中。

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201108184657.png)



union执行过程中，来自两条流的事件会以**FIFO的方式合并**，其**顺序无法保证**(**The operator does not**
**produce a specific order of events**.)。此外，union操作符**不会**对数据进行**去重**。每个输入事件都被发送到下游。



下面举个把三条数据流合并为一条的例子

```scala
val parisStream: DataStream[SensorReading] = ...
val tokyoStream: DataStream[SensorReading] = ...
val rioStream: DataStream[SensorReading] = ...
val allCities: DataStream[SensorReading] = parisStream,union(tokyoStream, rioStream)
```



##### 5.2.3.2 Connect，coMap，coFlatMap



考虑这样一个应用，它监视森林区域，并在发生火灾的风险很高时发出警报。应用从温度传感器和烟感传感器上接收数据。当**温度超过给定的阈值** **并且** **烟雾水平很高**时，应用程序会发出火灾警报。这时，为了判断两者是否同时成立，我们需要**合并两条流**来根据两条流的信息来**综合判断**



由此可见，合并两个流的事件是流处理中非常常见的需求。下面来看看相关的API



`DataStream.connect()`方法接收一个DataStream并返回一个ConnectedStreams对象，该对象表示两个联结在一起的流:

````scala
val first: DataStream[Int] = ...
val second: DataStream[String] = ...

val connected: ConnectedStreams[Int, String] = first.connect(second)
````



ConnectedStreams对象提供了map()和flatMap()，具体用法略



默认情况下，connect()不会在两个流的**事件之间** **建立关系**，因此两个流的事件被**随机分配**给算子任务。这种行为会产生**不确定的结果**，通常是不希望看到的。**为了在ConnectedStreams上产生确定性结果，可以将connect()与keyBy()或broadcast()结合使用。**

- 当使用`keyBy()`时，`connect()`转换会将两条数据流中具有**相同键的事件**发送到**同一个算子任务**上
- 而当使用`broadcast()`时，两条流中有一条被广播，它的事件被分发给下游算子的所有任务上。这样可以保证联合处理这两个输入流的元素。



##### 5.2.3.1 Split和Select



split是union的逆操作。它将**输入流** **分割**为与输入流相同类型的两个或**多个输出流**。**每个输入事件**可以被发送给**零个**、**一个**或**多个** **输出流**。因此，split操作还可以用于**过滤**或**复制**事件。



图5-6显示了一个split算子，它将所有白色事件与其他事件分开，发往不同的数据流。

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201108190954.png)



split()方法以一个OutputSelector函数接口作为参数。

````scala
// IN: 同DataStream的元素类型
OutputSelector[IN]
	> select(IN): Iterable[String]
````



每个输入事件到来时都会调用`OutputSelector.select()`方法，并随即返回一个`java.lang.Iterable[String]`。返回的这个`String`列表中的每个`String`是这个事件所属的输出流的名称。



`split()`方法返回一个`SplitStream`对象，这个对象提供一个`select()`方法，通过指定输出名称从SplitStream中选择一条或多条流。



实例5-2：将一个数字流分成一个大数字流和一个小数字流



```scala
val inputStream: DataStream[(Int, String)] = ...
val splitted: SplitStream[(Int, String)] = inputStream
	.split(t => if (t._1 > 1000) Seq("large") else Seq("small"))

val large: DataStream[(Int, String)] = splitted.select("large")
val small: DataStream[(Int, String)] = splitted.select("small")
val all: DataStream[(Int, String)] = splitted.select("large", "small")
```



#### 5.2.4 分发转换



当使用DataStream API构建应用程序时，系统会根据**操作语义**和配置的**并行度**来**自动选择数据分区策略**并将数据转发到正确的目标。有时我们可能希望能够**手动选择分区策略**。在本节中，我们将介绍DataStream中用于控制分区策略或自定义分区策略的方法。



下面是常见的分区策略

| 名称                  | 描述                                                         |
| --------------------- | ------------------------------------------------------------ |
| **随机**              | 随机数据交换策略由DataStream.shuffle()方法实现。该方法将事件随机分配到下游算子的并行任务中。 |
| **轮流**(Round-Robin) | 轮流方法将输入流的事件以轮流方式均匀分配给后继任务。         |
| **重调**(Rescale)     | rescale()也是以轮流的方式对事件进行分发，但是每个上游任务只与一部分下游任务建立发送通道。当上游任务数远小于下游任务数时，这种方法比较好用，下图展示了轮流和重调的区别![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201108193626.png) |
| **广播**(Broadcast)   | broadcast()将输入流中的事件复制，并发送给发送给下游算子的所有并行任务。 |
| **全局**(global)      | global()方法将**输入数据流的所有事件**发送给下游操作符的**第一个并行任务**。必须小心使用这种分区策略，因为将所有事件路由到同一任务可能会**影响应用程序性能** |
| **自定义**            | 如果所有预定义的分区策略都不合适，你可以利用partitionCustom()方法来自定义分区策略 |



自定义分区例子如下

```scala
val numbers: DataStream[(Int)] = ...

numbers.partitionCustom(myPartitioner, 0)

object myPartitioner extends Partitioner[Int]{
    val r = scala.util.Random
    
    override def partition(key: Int, numPartitions: Int): Int = {
        if (key < 0) 0 else r.nextInt(numPartitions)
    }
}
```



### 5.3 设置并行度



**算子的并行度**可以在**执行环境级别**或**算子级别**进行**控制**。**默认情况**下，应用的所有**算子的并行度**被设置为应用**执行环境的并行度**。而**执行环境的并行度**将**根据**应用**启动时所处**的上下文**自动初始化**。

- 如果应用程序在**本地执行环境**中运行，则将并行度设置为**与CPU内核数量相等**。
- 在向运行的**Flink集群**提交应用程序时，除非用户显示指定，否则环境并行度将设置为**集群的默认并行度**



最好将**算子并行度**设置为**随环境并行度变化的值**而不要设置为定值，例如：假设环境并行度为`x`，可以设置算子并行度为`y = x/2`。当在本机运行时`x=8, y=4`。而在集群运行时`x=32,y=16`。这样当运行环境变化时，算子并行度也可以随之变化。



下面的例子演示了如何获取环境并行度以及如何设置环境并行度

````scala
// 获取环境并行度
val env = StreamExecutionEnvironment.getExecutionEnvironment
val defaultParallelism = env.getParallelism

// 设置环境并行度
env.setParallelism(32)
````



下面的例子演示了如何设置算子的并行度

````scala
// 获取环境并行度
val env = StreamExecutionEnvironment.getExecutionEnvironment
val defaultParallelism = env.getParallelism

val result = env.addSource(new CustomSource)
	// 设置map的并行度为默认并行度的两倍
	.map(new MyMapper).setParallelism(defalutParallelism * 2)
	// print数据汇的并行度固定为2
	.print().setParallelism(2)
````



### 5.4 类型



Flink DataStream应用所处理的**事件**会**以数据对象的形式存在**。这些**数据对象**需要能够被**序列化**和**反序列化**，以通过网络发送它们，或将它们写入状态后端、检查点和保存点，或从状态后端读取。Flink使用**类型信息(**type information)的概念来表示数据类型，并为每种数据类型**自动生成**特定的**序列化器、反序列化器和比较器**。



一般情况下，Flink都可以**自动提取**数据对象的**类型信息**，但当自动提取器**失效**时，我们也需要**手动**指定类型信息。



本节我们会讨论Flink支持的类型，如何为数据类型创建类型信息，以及当Flink无法自动推断函数的返回类型的时如何以提示的方式帮助类型系统。



#### 5.4.1 支持的数据类型



Flink支持Java和Scala中可用的所有常见数据类型，可以分为以下类别

- **原始类型**
- Java和Scala**元组**
- Scala**样例类**(case class)
- **POJO**
- 一些特殊类型：数组、列表、映射、枚举等



对于**POJO**的解释：如果一个类满足如下条件，它会被Flink看作POJO

- 是一个公有类
- 有一个公有的无参默认构造函数
- 所有字段都是公有的或者提供了相应的`getter`以及`setter`方法
- 所有字段类型都必须是Flink支持的



对**特殊类型**的解释：Flink支持多种特殊类型，比如

- 原始或者对象类型的数组；
- Java的ArrayList、HashMap和Enum类型
- Hadoop的Writable类型。
- Scala的Either、Option和Try类型以及Flink内部实现的Java版本的Either类型



#### 5.4.2 为数据类型创建类型信息



在Flink的类型系统中，**核心类是`TypeInformation`**。它**为系统**生成**序列化器**和**比较器**提供了**必要的信息**。当应用提交执行时，Flink的类型系统尝试为框架处理的每个数据类型**自动推断**`TypeInformation`。因此，**大多数情况下**，我们都**没有必要手动**指定类型信息，但当自动推断**失灵**时，就需要我们为特定数据类型**手动**生成TypeInformation了。



下面举几个生成TypeInformation的例子

```scala
// 原始类型的TypeInformation
val stringType: TypeInformation[String] = Types.STRING
// Scala元祖的TypeInformation
val tupleType: TypeInformation[(Int, Long)] = Types.TUPLE[(Int, Long)]
// case class的TypeInformation
val caseClassType: TypeInformation[Person] = Types.CASE_CLASS[Person]
```



#### 5.4.3 显式提供类型信息



显示提供TypeInformation的方式有**两种**。**第一种**是通过**实现`ResultTypeQueryable`接口**来扩展函数。如下面例子所示

````scala
class Tuple2ToPersonMapper extends MapFunction[(String, Int), Person] with ResultTypeQueryable[Person] {
    override def map(v: (String, Int)): Person = Person(v._1, v._2)
    
    //实现ResultTypeQueryable
    override def getProducedType: TypeInformation[Person] = Types.CASE_CLASS[Person]
}
````



**第二种**，在定义Dataflow时使用Java DataStream API中的**`returns()`方法**来**显式指定**某算子的返回类型

```scala
persons = inputStream
	.map(t => new Person(t._1, t._2))
	.returns(Types.CASE_CLASS[Person])
```



### 5.5 定义键和引用字段



在Flink中有很多需要使用**键索引**(key specification)和**字段引用(**field reference)的地方。**Flink采用各种各样的灵活方式来定义键**：通过元素的字段位置来定义、通过基于字符串的字段表达式来定义、通过KeySelector函数来定义



#### 5.5.1 字段位置



如果数据类型是**元组**，则只需使用对应元组元素的**字段位置**就可以定义键。



例如下面这个例子使用**元组的第二个字段**作为**输入流的键值**

````scala
val input: DataStream[(Int, String, Long)] = ...
val keyed = input.keyBy(1)
````



此外，还可以使用多个元组字段来定义复合键

````scala
val keyed2 = input.keyBy(1, 2)
````



#### 5.5.2 字段表达式



另一种定义键和选择字段的方法是使用**基于字符串的字段表达式**。字段表达式适用于元组、pojo和case类。



```scala
// 最简单的字段表达式
val keyedSensors = sensorStream.keyBy("id")

// 在元组类型上使用字段表达式
val keyed = inputStream.keyBy("_1")

// 使用点运算符来嵌套POJO字段为键值
val persons = inputStream.keyBy("address.zip")

// 使用通配符 `_` 来选择元组中的全部字段作为键
val keyed = inputStream.keyBy("birthday._")
```



#### 5.5.3 KeySelector函数



第三种指定键的方式是使用**KeySelector函数**。它可以从输入事件中提取键

````scala
// T: 输入元素的类型
// KEY: 键值类型
KeySelector[IN, KEY]
	> getKey(IN): KEY
````



下面的例子会返回元组中的最大字段来作为键值

````scala
val input = DataStream[(Int, Int)] = ...
val keyedStream = input.keyBy(value => math.max(value._1, value._2))
````



### 5.6 实现函数



在DataStream API中有很多地方需要使用自定义函数。本节将介绍Flink中定义函数的几种方式



#### 5.6.1 函数类



Flink中所有**用户自定义函数**都是以**接口**或者**抽象类**的形式对外暴露的，如MapFunction、FilterFunction和ProcessFunction等



我们可以通过实现接口或者继承抽象类的方式来定义函数，例如下面的例子

```scala
class MyFilter extends FilterFunction[String]{
    override def filter(value: String): Boolean = {
        value.contains("flink")
    }
}

val filted = sentences.filter(new MyFilter())
```



> **函数必须是可序列化的**
>
> Flink使用Java序列化来序列化所有函数对象，以便将它们发送给对应的算子任务。用户函数中包含的所有内容都必须是可序列化的。如果您的函数需要一个非序列化的对象实例，您可以将其实现为一个富函数，并在open()方法中初始化非序列化字段，或者重写Java序列化和反序列化方法。



#### 5.6.2 Lambda函数



也可以通过Lambda表达式的方式来定义函数

```scala
val filted = sentences.filter(_.contains("flink"))
```



#### 5.6.3 富函数



有时，我们需要在**函数** **处理第一个记录之前**进行一些**初始化工作**或者取得**函数执行**相关的**上下文**信息。DataStream API提供了丰富的函数，它和我们之前见到的普通函数相比可以对外提供更多功能。



DataStream API中的所有转换函数都有对应的富函数，富函数的使用位置和普通函数以及Lambda函数相同。富函数的名称以Rich开头，例如RichMapFunction、RichFlatMapFunction等。



当使用富函数时，你可以对应函数的生命周期实现两个额外的方法：

- `open()`方法是富函数的**初始化方法**。它在每个任务首次调用转换方法之前调用一次
- `close()`方法是富函数的**终止方法**，会在每个任务**最后一次调用转换方法**后**调用一次**。因此，它通常用于**清理**和**释放**资源。

- 另外，还可以使用富函数自带的`getRuntimeContext()`方法来从函数的Runtime中获取一些信息



````scala
class MyFlatMap extends RichFlatMapFunction[Int, (Int, Int)]{
    var subTaskIndex = 0
    
    override def open(config: Configuration): Unit = {
        subTaskIndex = getRuntimeContext.getIndexOfThisSubtask
        //进行一些初始化工作
    }
    
    override def flatMap(in: Int, out: Collector[(Int, Int)]): Unit = {
        //子任务的编号从0开始
        if(in % 2 == subTaskIndex){
            out.collect((subTaskIndex, in))
        }
        //做一些额外处理工作
    }
    
    override def close(): Unit = {
        //做一些清理工作
    }
}
````





### 5.7 导入外部和Flink依赖



在实现Flink应用时经常需要添加一些外部依赖。应用在执行时，必须能够访问到所有依赖。**默认情况**下，Flink集群**只**加载**核心API依赖**(DataStream和DataSet API)，对于应用的**其他依赖**则必须**显式提供**。



有**两种方法**来确保所在执行应用时可以访问到所有依赖:

1. 将所有依赖打进应用的Jar包中，生成一个“胖Jar”
2. 将依赖放到Flink的`./lib`目录下，这样在Flink进程启动时就会将依赖加载到Classpath中



推荐使用第一种方式。