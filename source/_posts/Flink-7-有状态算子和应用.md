---
title: '[Flink][7][有状态算子和应用]'
date: 2020-11-15 18:56:30
tags:
    - 大数据
    - Flink
    - 流处理
categories:
    - Flink
---
## 第7章 有状态算子和应用



大多数复杂一点的算子都需要存储部分数据。Flink的许多**内置**的数据流**算子**、**数据源**和**数据汇**都是**有状态的**。例如：

- 窗口算子为ProcessWindowFunction**收集输入事件**或为ReduceFunction**保存聚合结果**
- **ProcessFunction**需要**保存计时器**
- 一些**数据汇函数**需要**维护事务**的状态

除了内置的算子、数据源、数据汇之外，Flink的DataStream API还提供了一些接口，用于在**用户自定义函数(user-defined function)**中**注册**、**维护**和**访问** **状态**。



本章重点介绍

1. **如何**在用户定义的函数中**定义不同类型的状态**并与之交互。
2. 我们还将讨论**性能**方面以及**如何控制函数状态的大小**。
3. 最后，我们将展示如何将键状态配置为**可查询状态**，以及如何从外部应用程序访问它。



### 7.1 实现有状态函数



函数有两种类型的状态：**键状态**(keyed state)和**算子状态**(operator state)。在本节中，我们将一一介绍如何实现具有键状态的函数和具有算子状态的函数。



#### 7.1.1 键状态



用户自定义函数可以在键属性的上下文中**存储**和**访问** **对应的键状态**。Flink会为**键域中**的**每个键**都维护**一个状态实例**，这些实例会分布在算子的那些并行任务上。也就是说函数所在算子的**每个并行任务**都负责**键域的一个子域**，并维护**子域上每个键** **对应**的那个**状态实例**。因此，键状态非常类似于**分布式键值映射**(distributed key-value map)。



**键状态**只能**应用在KeyedStream上**。



Flink为**键状态**提供了多个**状态原语**。状态原语定义了在**单个键**上**状态实例**的**存储结构**。Flink支持以下状态原语:



| 状态原语                   | 描述                                                         |
| -------------------------- | ------------------------------------------------------------ |
| **ValueState[T]**          | **保存**类型为T的**单个值**。可以通过`ValueState.value()`来**获取**，通过`ValueState.update(value: T)`来**更新** |
| **ListState[T]**           | 保存类型为T的元素**列表**。支持add、addAll、get等操作。但是**不支持对单个元素的删除**，但可以通过`update`方法来用新列表替换原来的列表 |
| **MapState[K, V]**         | 保存一个键和值的**映射**。该原语提供了许多Java Map接口的常规方法 |
| **ReducingState[T]**       | **用于聚合操作**，与ListState[T]的API大致相似。但调用它的add()会立即使用ReduceFunction聚合值。并且它的get()方法只会返回一个**单值列表**：这个**值**是**聚合结果** |
| **AggregatingState[I, O]** | **用于聚合操作**，比ReducingState更通用一点。调用它的add()会立即使用AggregateFunction聚合值 |



下面举个例子。如果传感器测量的温度自上次测量以来发生了超过阈值的变化，示例应用程序将发出警报事件。

```scala
object KeyedStateFunction {

  /** main() defines and executes the DataStream program */
  def main(args: Array[String]) {

    // 省略不重要代码
    val env = ...

    // 省略不重要代码
    val sensorData: DataStream[SensorReading] = ...
    
    // 先分key  
    val keyedSensorData: KeyedStream[SensorReading, String] = sensorData.keyBy(_.id)

    // 对keyedStream调用flatMap  
    val alerts: DataStream[(String, Double, Double)] = keyedSensorData
      .flatMap(new TemperatureAlertFunction(1.7))

    // print result stream to standard out
    alerts.print()

    // execute application
    env.execute("Generate Temperature Alerts")
  }
}

/**
  * The function emits an alert if the temperature measurement of a sensor changed by more than
  * a configured threshold compared to the last reading.
  *
  * @param threshold The threshold to raise an alert.
  */
class TemperatureAlertFunction(val threshold: Double)
    extends RichFlatMapFunction[SensorReading, (String, Double, Double)] {

  // 用来存储 上一次温度 的状态      
  // the state handle object
  private var lastTempState: ValueState[Double] = _

  // 初始化工作      
  override def open(parameters: Configuration): Unit = {
    
    // create state descriptor 创建一个状态描述符
    val lastTempDescriptor = new ValueStateDescriptor[Double](
        "lastTemp", classOf[Double])
    
    // obtain the state handle 初始化并获取 上一次温度 状态的引用
    lastTempState = getRuntimeContext.getState[Double](lastTempDescriptor)
  }

  // 真正的处理函数      
  override def flatMap(reading: SensorReading,
  	out: Collector[(String, Double, Double)]): Unit = {

    // fetch the last temperature from state 从状态中拿到上一次的温度
    val lastTemp = lastTempState.value()
    // check if we need to emit an alert 计算差值
    val tempDiff = (reading.temperature - lastTemp).abs
    // 如果差值超过阈值，就会被输出
    if (tempDiff > threshold) { 
      // temperature changed by more than the threshold
      out.collect((reading.id, reading.temperature, tempDiff))
    }

    // update lastTemp state 更新状态
    this.lastTempState.update(reading.temperature)
  }
}

```



下面对例子中的几个重点进行解释

- 要**创建**一个**状态对象**，我们**必须**通过RichFunction的RuntimeContext向Flink**注册**一个**状态描述符**。

  - **每种状态原语**都有自己**特定的状态描述符**，描述符中包括**状态的名称**和**状态的数据类型**。例如，`val lastTempDescriptor = new ValueStateDescriptor[Double]("lastTemp", classOf[Double])`中，状态原语ValueState有自己的状态描述符类`ValueStateDescriptor`，我们通过提供状态的名称`lastTemp`和状态的类型`Double`来实例化状态描述符类。
  - **ReducingState**和**AggregatingState**的**描述符**在实例化时还需要**额外提供** **ReduceFunction或AggregateFunction**来对加入该状态的值进行聚合。
- 通过注册多个状态描述符，让处理函数拥有多个状态对象
- 因为Flink需要创建合适的**序列化器**，所以描述符中**必须指定数据类型**。（如前面类型指定的`classOf[Double]`）
- 一般把**状态引用**声明为类中的**成员变量**。然后，**状态引用**会在**open()方法**中被**初始化**。例如，前面例子的`lastTempState`这个状态被声明为了类的成员变量，并且在`open()`方法中`lastTempState = getRuntimeContext.getState[Double](lastTempDescriptor)`，它被初始化了。



当一个函数注册一个状态描述符时，Flink会**检查状态后端**中**是否已经**有该状态描述符对应的状态。在**从故障恢复算子**或者**从保存点启动应用**时，可能会**发生这种检查**。在这两种情况下，Flink都将**新注册**的状态引用**链接到**状态后端中**描述符相同**的**现存状态**。如果**找不到**，就将状态初始化为**空**。



此外，FlinkAPI还提供了一种更简洁的写法，我们用这种简便写法实现与上例逻辑相同的功能。

```scala
val alerts: DataStream[(String, Double, Double)] = keyedData
  .flatMapWithState[(String, Double, Double), Double] {
    // 处理第一个事件到达的情况（这时没法对比阈值差距）
    // 第一个参数是当前事件
    // 第二个参数是当前状态(Flink会从后端中取出状态并填充到这里)
    // 第一个返回值是flatMap的结果列表
    // 第二个返回值是处理后的状态(Flink会用这个值来对后端中的状态进行更新)  
    case (in: SensorReading, None) =>{
      // no previous temperature defined; just update the last temperature
      (List.empty, Some(in.temperature))
    }
    
    // 处理非第一个事件到达的情况（正常的处理，这里才能比较阈值）  
    case (r: SensorReading, lastTemp: Some[Double]) => {
      // compare temperature difference with threshold
      val tempDiff = (r.temperature - lastTemp.get).abs
      if (tempDiff > 1.7) {
        // threshold exceeded; emit an alert and update the last temperature
        (List((r.id, r.temperature, tempDiff)), Some(r.temperature))
      } else {
        // threshold not exceeded; just update the last temperature
        (List.empty, Some(r.temperature))
      }
    }
}
```



#### 7.1.2 算子状态



**每个算子的并行任务**管理**算子状态**。在算子的同一并行任务中处理的所有事件都可以访问相同的状态。Flink支持三种算子状态：列表状态、联合列表状态以及广播状态。



**处理函数**可以通过**实现`ListCheckpointed`接口**来**处理算子列表状态**。处理函数会将**算子状态**当作**常规的成员变量**来实现，然后**通过**ListCheckpoint接口中的两个**回调函数**与**状态后端** **交互**:



下面来看看ListCheckpointed接口的源码

````java
public interface ListCheckpointed<T extends Serializable> {

	// 以列表的形式返回一个函数状态的快照
	List<T> snapshotState(long checkpointId, long timestamp) throws Exception;

	// 根绝提供的列表来恢复函数状态
	void restoreState(List<T> state) throws Exception;
}
````

下面我们举个例子，它显示了如何为一个函数实现ListCheckpoint接口，该函数为算子的每个并行任务统计温度超过阈值的数目

```scala
class HighTempCounter(val threshold: Double)
    extends RichFlatMapFunction[SensorReading, (Int, Long)]
    with ListCheckpointed[java.lang.Long] {
  // 获取当前任务的索引标志
  private lazy val subtaskIdx = getRuntimeContext.getIndexOfThisSubtask
  // 本地计数器（当前任务的本地状态）
  private var highTempCnt = 0L
  // 处理函数，每当温度超过阈值就对本地计数器+1      
  override def flatMap(
      in: SensorReading, 
      out: Collector[(Int, Long)]): Unit = {
    if (in.temperature > threshold) {
      // increment counter if threshold is exceeded
      highTempCnt += 1
      // emit update with subtask index and counter
      out.collect((subtaskIdx, highTempCnt))
    }
  }
 /**
 * 用来返回快照，这个函数会在生成检查点时被调用
 * @Param chkpntId: 检查点编号
 * @Param ts: JobManager创建检测点时的时间戳
 * @Return 将本地状态转换为一个列表来返回
 */       
  override def snapshotState(
      chkpntId: Long, 
      ts: Long): java.util.List[java.lang.Long] = {
    // snapshot state as list with a single count
    java.util.Collections.singletonList(highTempCnt)
  }
 /**
 * 用于恢复本地状态，这个函数在初始化函数状态时被调用
 * @Param state: 将这个列表转换为本地状态
 */  
  override def restoreState(
      state: util.List[java.lang.Long]): Unit = {
    highTempCnt = 0
    // restore state by adding all longs of the list
    for (cnt <- state.asScala) {
      highTempCnt += cnt
    }
  }
}
```



##### 7.1.2.1 为什么要把算子状态当作列表来处理呢？



看完上个示例，您可能想知道为什么算子状态作为**状态对象列表(a list of state objects)**处理。这是因为**List结构** **支持改变**带算子状态的函数的**并行性**。为了增加或减少具有算子状态的函数的并行性，需要将算子状态重新分配给更多或更少的任务实例。这需要**分割**或**合并** **状态**对象，而一般来说，**列表**要比单个值**更加适合** **分割**和**合并**



通过提供状态对象列表，具有算子状态的函数可以使用`snapshotState()`和`restoreState()`方法实现此逻辑。

- `snapshotState()`方法将本地算子状态**分解**为多个部分
- `restoreState()`方法将本地算子状态从多个部分**组合**起来
- 当一个算子状态被恢复时，该状态的各个部分被分配到算子的所有**并行任务**中，并调用restoreState()方法。
- 如果**并行任务**比**状态对象** **多**，那么**有些任务启动时分配不到状态**，那么传入restoreState()方法时，**入参**就是个**空列表**。



那么，我们前面那个例子只是`java.util.Collections.singletonList(highTempCnt)`，这可能在增加并行度时导致有的任务获取不到状态，所以我们改造一下这个方法，如下代码

````scala
// 分割操作符列表状态，以便在重新缩放期间更好地分布
override def snapshotState(
    chkpntId: Long, 
    ts: Long): java.util.List[java.lang.Long] = {
  
  // split count into ten partial counts
  val div = highTempCnt / 10
  val mod = (highTempCnt % 10).toInt
  // 把这个本地计数器拆分成10份，返回一个长度为10的列表	
  // return count as ten parts
  (List.fill(mod)(new java.lang.Long(div + 1)) 
   	++ List.fill(10 - mod)(new java.lang.Long(div))).asJava
}
````



#### 7.1.3 使用连接广播状态(Using Connected Broadcast State)



流应用中的一个常见需求是将**相同的信息** **分发**到一个算子的**所有并行任务**中，并将其作为可恢复状态进行维护。



> 例如，一个规则流和一个依赖于这个规则的事件流。函数 connect 这两个输入流。它需要使用规则来处理所有的输入事件。因此我们需要对规则流进行广播，来让所有处理事件流的任务都接收到规则，都用这个规则来处理输入事件。



下面还是直接看个例子，看看如何实现用**广播流**来**动态配置阈值**的温度报警系统

```scala
object BroadcastStateFunction {

  /** main() defines and executes the DataStream program */
  def main(args: Array[String]) {

    // set up the streaming execution environment
    val env = ...

    // ingest sensor stream
    val sensorData: DataStream[SensorReading] = 

    // define a stream of thresholds
    // 定义一个阈值的流（规则流，需要广播，这样可以动态配置规则）  
    val thresholds: DataStream[ThresholdUpdate] = env.fromElements(
      ThresholdUpdate("sensor_1", 5.0d),
      ThresholdUpdate("sensor_2", 0.9d),
      ThresholdUpdate("sensor_3", 0.5d),
      ThresholdUpdate("sensor_1", 1.2d),  // update threshold for sensor_1
      ThresholdUpdate("sensor_3", 0.0d))  // disable threshold for sensor_3

    // 对事件流进行分key  
    val keyedSensorData: KeyedStream[SensorReading, String] = sensorData.keyBy(_.id)

    // 创建一个广播状态描述符（一个键状态描述符）
    // 名字是"thresholds"
    // 键的类型是String
    // 值的类型是Double  
    val broadcastStateDescriptor = new MapStateDescriptor[String, Double](
        "thresholds",
        classOf[String],
        classOf[Double]
    )
    
    // 使用一个或多个广播状态描述符来创建一个广播流  
    val broadcastThresholds: BroadcastStream[ThresholdUpdate] = thresholds
      .broadcast(broadcastStateDescriptor)

    // 将正常流与广播流联结，并且使用处理函数处理  
    val alerts: DataStream[(String, Double, Double)] = keyedSensorData
      .connect(broadcastThresholds)
      .process(new UpdatableTemperatureAlertFunction())

    // print result stream to standard out
    alerts.print()

    // execute application
    env.execute("Generate Temperature Alerts")
    }
}

case class ThresholdUpdate(id: String, threshold: Double)

/**
  * 处理函数需要实现KeyedBroadcastProcessFunction接口，它有四个类型参数
  * The key type of the input keyed stream.
  * 事件流中元素的类型(The input type of the keyed (non-broadcast) side.)
  * 广播流中元素的类型(The input type of the broadcast side.)
  * 输出的类型
  */
class UpdatableTemperatureAlertFunction()
    extends KeyedBroadcastProcessFunction[String,
                                          SensorReading,
                                          ThresholdUpdate,
                                          (String, Double, Double)] {

  // 阈值状态描述符      
  private lazy val thresholdStateDescriptor 
        = new MapStateDescriptor[String, Double]
        ("thresholds", classOf[String], classOf[Double])

  // ValueState[Double]状态 用来储存上一个温度的
  private var lastTempState: ValueState[Double] = _

  // 初始化函数      
  override def open(parameters: Configuration): Unit = {
    // 创建上一个温度状态的描述符
    val lastTempDescriptor 
      = new ValueStateDescriptor[Double]("lastTemp", classOf[Double])
    // 根据描述符来初始化上一个温度状态
    lastTempState = getRuntimeContext.getState[Double](lastTempDescriptor)
  }

  // 处理广播流的事件      
  override def processBroadcastElement(
      update: ThresholdUpdate,
      ctx: KeyedBroadcastProcessFunction[
          String,
          SensorReading,
          ThresholdUpdate,
          (String, Double, Double)]#Context,
      out: Collector[(String, Double, Double)]): Unit = {

    // 从上下文获取广播状态  
    val thresholds = ctx.getBroadcastState(thresholdStateDescriptor)

    // 如果传入状态的阈值不等于0.0d，更新广播状态  
    if (update.threshold != 0.0d) {
      thresholds.put(update.id, update.threshold)
    } else {
      // 如果传入的阈值为0.0d，就说明用户不想设置阈值了，就直接删掉广播状态中对应键
      thresholds.remove(update.id)
    }
  }

  // 处理正常流的事件   
  override def processElement(
      reading: SensorReading,
      readOnlyCtx: KeyedBroadcastProcessFunction[
          String,
          SensorReading,
          ThresholdUpdate,
          (String, Double, Double)]#ReadOnlyContext,
      out: Collector[(String, Double, Double)]): Unit = {

    // 获得只读的广播状态
    val thresholds = readOnlyCtx.getBroadcastState(thresholdStateDescriptor)
    // 获取key对应的阈值，然后比较是否超过阈值，超过就报警
    if (thresholds.contains(reading.id)) {
      // get threshold for sensor
      val sensorThreshold: Double = thresholds.get(reading.id)

      // fetch the last temperature from state
      val lastTemp = lastTempState.value()
      // check if we need to emit an alert
      val tempDiff = (reading.temperature - lastTemp).abs
      if (tempDiff > sensorThreshold) {
        // temperature increased by more than the threshold
        out.collect((reading.id, reading.temperature, tempDiff))
      }
    }

    // 更新上一个温度
    this.lastTempState.update(reading.temperature)
  }
}
```



对于上面的示例，有几点要说明一下

- KeyedBroadcastProcessFunction的元素处理方法是**不对称**的。方法processElement()和processBroadcastElement()参数中的上下文**一个支持读写**，**一个只可读**。



#### 7.1.4 使用CheckpointedFunction接口



**CheckpointedFunction接口**是**指定有状态函数**的**最底层接口**。它提供了**钩子方法**来**注册**和**维护** **键状态**和**算子状态**，并且是唯一允许访问算子列表联合状态(在恢复或保存点重新启动时完全复制的操作符状态)的接口。



CheckpointedFunction接口定义了两个方法，initializeState()和snapshotState()。

- **initializeState()**方法在**创建**CheckpointedFunction的**并行任务时**被调用。当启动应用程序或由于故障而重新启动任务时，就会创建并行任务从而调用这个方法。该方法用于**初始化**状态或者**恢复**状态

- 在**生成检查点之前**，**snapshotState()**方法被调用，snapshotState()方法的目的是确保**在检查点生成完成之前** **更新**所有**状态对象**（这样检查点拿到的本地任务状态就是新的）。



下面还是举个例子，这个例子**同时具有键状态和算子状态**，它统计每个键和每个算子任务有多少传感器读数超过了指定的阈值。

```scala
class HighTempCounter(val threshold: Double)
  extends FlatMapFunction[SensorReading, (String, Long, Long)]
	// 需要实现CheckpointedFunction接口		
    with CheckpointedFunction {

  // 本地变量，用来统计当前任务超过阈值的温度个数
  var opHighTempCnt: Long = 0

  // 键状态，用来储存当前key超过阈值的温度个数      
  var keyedCntState: ValueState[Long] = _
        
  // 算子状态，用来储存当前任务超过阈值的温度个数 
  var opCntState: ListState[Long] = _

  // 处理函数      
  override def flatMap(
      v: SensorReading, 
      out: Collector[(String, Long, Long)]): Unit = {
    if (v.temperature > threshold) {
      // 本地变量++
      opHighTempCnt += 1
      // 更新键状态
      val keyHighTempCnt = keyedCntState.value() + 1
      keyedCntState.update(keyHighTempCnt)

      // emit new counters 输出
      out.collect((v.id, keyHighTempCnt, opHighTempCnt))
    }
  }

  // 初始化      
  override def initializeState(initContext: FunctionInitializationContext): Unit = {
    // initialize keyed state 初始化键状态
    val keyCntDescriptor = new ValueStateDescriptor[Long]("keyedCnt", classOf[Long])
    keyedCntState = initContext.getKeyedStateStore.getState(keyCntDescriptor)

    // initialize operator state 初始化算子状态
    val opCntDescriptor = new ListStateDescriptor[Long]("opCnt", classOf[Long])
    opCntState = initContext.getOperatorStateStore.getListState(opCntDescriptor)
    // initialize local variable with state 用算子状态来设置本地变量的值
    opHighTempCnt = opCntState.get().asScala.sum
  }

  // 快照（用于检查点前的更新）      
  override def snapshotState(snapshotContext: FunctionSnapshotContext): Unit = {
    // update operator state with local state 把本地变量储存的值更新到算子状态
    opCntState.clear()
    opCntState.add(opHighTempCnt)
  }
}
```



#### 7.1.5 接收检查点完成的通知



由于检查点机制，Flink可以实现非常好的性能。然而，另一个含义是，**除了生成检查点时的几个逻辑时间点，应用永远不会处于一致的状态**。**对于一些算子**来说，**知道检查点是否完成**是很**重要**的。（例如，旨在将数据写入具有严格一致性要求的外部系统的数据汇必须只发出在检查点完成之前接收到的记录，以确保在出现故障的情况下不会重新发送数据。）



**只有所有算子任务的状态都成功地写入检查点存储**时，检查点**才算成功**。因此，**只有JobManager**才**能确定**检查点是否成功。



需要接收检查点完成的通知的算子可以**实现CheckpointListener接口**。这个接口提供了notifyCheckpointComplete(long  chkpntId)方法，当JobManager确定一个检查点成功完成后，该方法会被调用。



### 7.2 为有状态的应用开启故障恢复



Flink通过**检查点机制**来启动故障恢复，我们**需要显式地启动**检查点机制，如下所示

````scala
val env = StreamExecutionEnvironment.getExecutionEnvironment

// set checkpointing interval to 10 seconds (10000 milliseconds)
env.enableCheckpointing(10000L)
````



**检查点间隔**(如前文的10s)会**影响**检查点机制在**常规处理期间的开销**以及**从故障中恢复所需的时间**。较短的检查点间隔在常规处理期间会导致较高的开销，但可以实现更快的恢复，因为需要重新处理的数据更少。



Flink提供了其他一些可供调节的配置选项，比如

- **一致性保障**(精确一次或至少一次)的选择
- **并发检查点的数量**
- 用来取消长时间运行检查点的**超时时间**
- 和**状态后端相关**的选项。



### 7.3 确保有状态应用的可维护性



已经运行了几周的应用的**状态**可能**代价很高**，甚至无法重新计算。同时，长时间运行的应用程序也需要一些维护，比如修复bug、添加、调整或删除功能，或者调整算子的并行性等。因此，能够将状态**迁移到新版本**应用或**重分配**到更多或更少的算子任务是很重要的。



Flink提供**保存点**来**维护**应用**状态**。但是，它要求**初始版本的应用**的全部有状态**算子**都**指定两个参数**，以确保将来可以正确地维护应用程序。

1. 算子的**唯一标识符**
2. **最大并行度**(只针对基于键的算子)。



**算子**的**唯一标识符**和**最大并行度**被**固化**到保存点中，不能更改。如果标识符或最大并行度被更改，则**不能**从以前的保存点**重启**应用。



#### 7.3.1 指定算子唯一标识



应该为应用的每个算子指定唯一标识符。这个标识符是保存点中的元数据。当从保存点启动应用程序时，标识符用于将保存点中的状态映射到已启动应用的相应算子。只有当已启动应用程序的算子的标识符相同时，才能将保存点状态恢复到它们。



设置方式如下，强烈建议手动设置

```scala
val alerts: DataStream[(String, Double, Double)] = keyedSensorData
  .flatMap(new TemperatureAlertFunction(1.1))
  // uid方法，用来设置并行度	
  .uid("TempAlert")
```



#### 7.3.2 为使用键状态的算子定义最大并行度



算子的**最大并行度**参数定义了算子在对键状态进行分割时，所用到的**键值组数目**。该数量限制了键状态的**算子**可以被扩展到的**最大并行任务数**。（因为**一个并行任务** **至少**要有**一个键值组**）



下面展示如何设置最大并行度

````scala
val env = StreamExecutionEnvironment.getExecutionEnvironment

// 为应用设置最大并行度
env.setMaxParallelism(512)

// 为算子设置最大并行度
val alerts: DataStream[(String, Double, Double)] = keyedSensorData
  .flatMap(new TemperatureAlertFunction(1.1))
  // set the maximum parallelism for this operator and
  // override the application-wide value
  .setMaxParallelism(1024)
````



如果**算子**在首个版本**没有设置最大并行度**，则会根据首个版本的并行度来**推断**它

- 如果并行度小于等于128，则最大并行度为128。
- 如果该操作符的并行度大于128，则最大并行度计算为`Min(nextPowerOfTwo(parallelism +  (parallelism / 2)),2^15)`。



### 7.4 有状态应用的性能及鲁棒性



算子与状态的交互会影响应用程序的**健壮性**(robustness)和**性能**(perfermance)。比如选择状态后端、检查点算法的配置、状态的大小都会影响健壮性和性能。



#### 7.4.1 选择状态后端



**状态后端**负责**存储** **每个任务**的**本地状态**，并在**执行检查点时**将其持久化到**远程存储**。由于本地状态可以以不同的方式进行维护和检查点，状态后端是**可插拔的(pluggable)**。**不同的应用**可以**选择** **不同的状态后端**实现来维护它们的状态。状态后端的选择对有状态应用程序的健壮性和性能有影响。**每种状态后端**都**提供**了**状态原语的实现**，比如ValueState、ListState和MapState。



Flink提供了三种状态后端

- MemoryStateBackend
- FsStateBackend
- RocksDBStateBackend



##### 7.4.1.1 MemoryStateBackend(内存式的，易丢但快)



MemoryStateBackend将状态作为**常规对象**存储在TaskManager  **JVM进程的堆**上。

- 例如，MapState由Java  HashMap对象支持。
- 虽然这种方法提供了非常低的读写延迟，但它对应用程序的健壮性有影响。
  - 如果任务实例的状态变得太大，**JVM**和在其上运行的所有任务实例可能会由于**OutOfMemoryError**而**被杀死**。
  - 此外，这种方法可能会出现**垃圾回收暂停**(garbage collection pause)问题，因为它将许多长期存在的对象放在堆内存上。
- 当**生成检查点**时，MemoryStateBackend将状态发送给JobManager,  **JobManager**将其**存储在堆内存中**。因此，应用程序的总状态必须符合JobManager的内存。因为它的内存是易失性的，所以在JobManager失败的情况下会丢失状态。
- 由于这些限制，MemoryStateBackend仅推荐用于**开发**和**调试**目的。



##### 7.4.1.2 FsStateBackend(本地状态在内存，检查点会持久化)



- FsStateBackend在TaskManager的**JVM**堆上**存储本地状态**，就像MemoryStateBackend一样。
- 然而，FsStateBackend方式下，**检查点**会被写入**远程持久文件系统**。
- FsStateBackend提供了**内存读写速度级别**的**本地访问**和**持久化级别**的**故障容错**。
- 但是，它受到TaskManager内存大小的限制，可能会出现垃圾收集暂停。



##### 7.4.1.3 RocksDBStateBackend（本地持久化，检查点也持久化）



- RocksDBStateBackend将所有**本地状态**存储到**本地RocksDB实例中**。
- RocksDB是一个嵌入式键值存储，它**将数据持久化到本地磁盘**。为了从RocksDB读写数据，需要进行序列化和反序列化。在**生成检查点**时，RocksDBStateBackend还将状态发送到**远程持久文件系统**。
- 所以对于具有非常大状态的应用程序，RocksDBStateBackend是一个不错的选择。



下面展示如何为应用配置状态后端

````scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
val checkpointPath: String = ???
// new 一个RocksDB状态后端实例
val backend = new RocksDBStateBackend(checkpointPath)
// configure the state backend 配置它
env.setStateBackend(backend)
````



#### 7.4.2 选择状态原语



因为RocksDB等后端的状态**读写**涉及**序列化**和**反序列化**，所以**状态原语的选择**很**影响算子性能**。例如：

- ValueState在被访问时是完全反序列化，在被更新时会完全序列化。
- RocksDBStateBackend的ListState在**读取值**之前对**所有列表条目进行反序列化**。但是，向ListState**添加单个值**是一种**廉价的操作**，因为只有追加的值需要序列化，而整个列表不需要完全序列化。
- MapState的RocksDBStateBackend允许读写在键的级别上提供序列化控制。（可以只序列化或者反序列化某个键及其对应的值）。



举例子，

- 针对RocksDBStateBackend来说，使用`MapState[X,Y]`要比`ValueState[HashMap[X,Y]]`更高效。

- 如果经常在列表尾部添加元素，但很少访问列表，那么`ListState[X]`要比`ValueState[List[X]]`更高效。



此外，建议对于同一个状态来说，每次函数调用只更新一次状态。



#### 7.4.3 防止状态泄露



流应用程序通常设计为连续运行数月或数年。如果**应用状态** **不断增加**，在**某个时刻**它会变得太大并**杀死应用程序**，除非采取措施为应用提供更多的资源。为了**防止应用的资源消耗随着时间而增加**，**控制算子状态的大小**非常重要。由于状态的处理直接影响算子语义，所以**Flink不能自动清除状态**并释放存储空间。相反，**所有有状态算子都必须承担控制其状态的大小的责任，确保它不会无限增长。**



**状态无限增长**的一个常见**原因**是**键域无限增长**。

>  例如，在点击事件流，点击事件有一个session_id属性，该属性在一段时间后失效。在这种情况下，具有键状态的函数将积累越来越多键的状态。**随着键域的增大，状态不断增大，但是过期键的状态是陈旧而无用**。这个问题的解决方案是**删除过期键**。
>
>  并且这种情况也会发生在部分DataStreamAPI的**内置算子**上，比如：针对KeyedStream的那些内置聚合函数，`min`、`max`、`sum`等等。所以，在使用这些内置算子时，一定要**保证键域不是无限增加**的



我们可以通过注册计时器，并声明回调函数的方式来清理过期的键



下面举个例子。它会清除在一小时内没有提供任何新的温度测量值的键。

```scala
object StatefulProcessFunction {

  /** main() defines and executes the DataStream program */
  def main(args: Array[String]) {
    
    // set up the streaming execution environment
    val env = ...

    // ingest sensor stream
    val sensorData: DataStream[SensorReading] = ...

    val keyedSensorData: KeyedStream[SensorReading, String] = sensorData.keyBy(_.id)

    val alerts: DataStream[(String, Double, Double)] = keyedSensorData
      .process(new SelfCleaningTemperatureAlertFunction(1.5))

    // print result stream to standard out
    alerts.print()

    // execute application
    env.execute("Generate Temperature Alerts")
  }
}


class SelfCleaningTemperatureAlertFunction(val threshold: Double)
    extends KeyedProcessFunction[String, SensorReading, (String, Double, Double)] {

  // 状态，用来保存上一个温度
  private var lastTempState: ValueState[Double] = _
  
  // 状态，用来保存上一个计时器的时间点      
  private var lastTimerState: ValueState[Long] = _

  override def open(parameters: Configuration): Unit = {
    // 注册并初始化上一个温度状态
    val lastTempDescriptor = new ValueStateDescriptor[Double]("lastTemp",
    	classOf[Double])
    
    lastTempState = getRuntimeContext.getState[Double](lastTempDescriptor)
    
    // 注册并初始化上一个计时器状态
    val timestampDescriptor: ValueStateDescriptor[Long] 
      = new ValueStateDescriptor[Long](
          "timestampState", classOf[Long])
    
    lastTimerState = getRuntimeContext.getState(timestampDescriptor)
  }

  // 处理函数      
  override def processElement(
      reading: SensorReading,
      ctx: KeyedProcessFunction[String, SensorReading, (String, Double, Double)]#Context,
      out: Collector[(String, Double, Double)]): Unit = {

    // 计算新计时器的触发时间
    val newTimer = ctx.timestamp() + (3600 * 1000)
    // 获取当前计时器
    val curTimer = lastTimerState.value()
    // 删除当前计时器然后注册新计时器
    ctx.timerService().deleteEventTimeTimer(curTimer)
    ctx.timerService().registerEventTimeTimer(newTimer)
    // 更新计时器触发时间到lastTimerState状态上
    lastTimerState.update(newTimer)

    // 获取上一个温度，比较然后决定是否发出警报
    val lastTemp = lastTempState.value()
    val tempDiff = (reading.temperature - lastTemp).abs
    if (tempDiff > threshold) {
      out.collect((reading.id, reading.temperature, tempDiff))
    }

    // 更新上一个温度
    this.lastTempState.update(reading.temperature)
  }

  // 计时器到时间时，这个函数被触发      
  override def onTimer(
      timestamp: Long,
      ctx: KeyedProcessFunction[String, SensorReading, (String, Double, Double)]#OnTimerContext,
      out: Collector[(String, Double, Double)]): Unit = {

    // 清除一个小时都没有收到事件的键对应的状态
    lastTempState.clear()
    lastTimerState.clear()
  }
}
```



### 7.5 更新有状态应用



对长时间运行的**有状态流应用**进行**错误修复**或**业务逻辑更改**常常发生。因此，我们要保证在流应用发生**版本更新**时，**不丢失应用状态**



Flink通过三个步骤来实现版本更新

1. 为正在运行的应用**生成保存点**
2. **停止老版本**应用
3. **从保存点启动新版本**的应用。



但是只有新老版本的**保存点兼容**时，才能完成更新。也就是说，Flink只支持下面三种更新（只有这三种更新的保存点是兼容的）

- 在**不更改**或删除现有**状态**的情况下**更改**应用的**逻辑**。这包括向应用中添加算子。
- 从应用中**删除某个状态**。
- 通过**更改状态原语类型或状态的数据类型**来修改现有算子的状态（只有部分情况可以兼容）



#### 7.5.1 保持现有状态更新应用



如果应用**在没有删除或更改现有状态的情况下**进行了更新，那么它始终是与保存点**兼容**的，并且可以从老版本的保存点启动。



如果向应用**添加**新的有状态**算子**，或向现有**算子**添加新的**状态**，则在从保存点启动应用时，**该状态将初始化为空**。（新添加的算子或状态，启动时初始化为空）



#### 7.5.2 从应用中删除状态



还可以通过**删除状态**来调整应用程序。可以**删除完整的算子**或仅从算子中**删除一个状态**。这种情况下，保存点的部分状态将无法映射到新版本应用。



默认情况下，Flink将**不会启动** **没有恢复**保存点中包含的**所有状态**的应用程序，以避免丢失保存点中的状态。但是，可以禁用此安全检查



#### 7.5.3 修改算子的状态



增删状态比较容易兼容，但是修改状态很难做到兼容。



一般来说，会发生两种情况的修改

- **更改状态的数据类型**，例如将ValueState[Int]更改为ValueState[Double]，
- **更改状态原语的类型**，例如，将ValueState[List[String]]更改为ListState[String]



对于这两种情况，Flink会进行如下处理

- Flink目前还**不支持更改状态原语的类型**
- 由于涉及到序列化和反序列化机制。对于更改状态的数据类型：在Flink 1.7中，如果数据类型被定义为Apache  Avro类型，并且新数据类型也是根据Avro的模式演化规则从原始类型演变而来的Avro类型，那么支持更改状态的数据类型。



### 7.6 可查询式状态(queryable state)



许多流处理应用需要与其他应用共享它的结果。Apache Flink提供了**可查询状态**的特性来**与其他应用共享结果**。在Flink中，任何键状态都可以作为可查询状态以只读的形式暴露给外部应用程序。



#### 7.6.1 可查询式状态服务的架构及启动方式



Flink的可查询状态服务由三个组件组成:

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201113134348.png)

- **QueryableStateClient**：**外部应用** **使用**QueryableStateClient来提交查询和获取结果。
- **QueryableStateClientProxy**：QueryableStateClientProxy**接受并响应QSClient的请求**。每个TaskManager上都运行一个客户端代理。因为状态分布在各个TaskManager，因此ClientProxy先询问JobManager来得知需要查询的状态在哪个TaskManager上，然后向这个TaskManager的QSServer发送请求。
- **QueryableStateServer**：QueryableStateServer响应客户端代理的请求。每个TaskManager都运行一个StateServer，该Server**从本地状态后端获取键状态**，并将其返回给QSClientProxy。



#### 7.6.2 对外暴露可查询式状态



实现一个**带有可查询式状态**的**流应用**很容易。你要做的就是定义一个带有键状态的函数，并在获得状态引用之前，调用setQueryable(String)方法使状态变成可查询的。如下例所示



```scala
override def open(parameters: Configuration): Unit = {
   
   // 创建状态描述符
   val lastTempDescriptor = new ValueStateDescriptor[Double]("lastTemp", classOf[Double])
   
   // 启动可查询状态，并设置查询标识符
   lastTempDescriptor.setQueryable("lastTemperature")
   
   // 注册并初始化状态
   lastTempState = getRuntimeContext.getState[Double](lastTempDescriptor)
}
```



除此之外，Flink还**支持利用数据汇**将流中的**所有事件**都**存到可查询状态**中

```scala
val tenSecsMaxTemps: DataStream[(String, Double)] = sensorData
  // project to sensor id and temperature
  .map(r => (r.id, r.temperature))
  // compute every 10 seconds the max temperature per sensor
  .keyBy(_._1)
  .timeWindow(Time.seconds(10))
  .max(1)

// store max temperature of the last 10 secs for each sensor 
// in a queryable state
tenSecsMaxTemps
  // key by sensor id
  .keyBy(_._1)
  .asQueryableState("maxTemperature")
```



#### 7.6.3 从外部系统查询状态



**任何基于jvm的应用**都可以**使用QueryableStateClient** **查询正在运行的Flink应用**的**可查询状态**。



下面举个例子

```scala
object TemperatureDashboard {
  // assume local setup and TM runs on same machine as client
  val proxyHost = "127.0.0.1"
  val proxyPort = 9069
  // jobId of running QueryableStateJob
  // can be looked up in logs of running job or the web UI
  val jobId = "d2447b1a5e0d952c372064c886d2220a"
  // how many sensors to query
  val numSensors = 5
  // how often to query the state
  val refreshInterval = 10000
  def main(args: Array[String]): Unit = {
    // configure client with host and port of queryable state proxy
    val client = new QueryableStateClient(proxyHost, proxyPort)
    val futures = new Array[
      CompletableFuture[ValueState[(String, Double)]]](numSensors)
    
    val results = new Array[Double](numSensors)
    // print header line of dashboard table
    
    val header = (for (i <- 0 until numSensors) yield "sensor_" + (i + 1)
    	.mkString("\t| ")
    
    println(header)
    // loop forever
    while (true) {
      // send out async queries
      for (i <- 0 until numSensors) {
        futures(i) = queryState("sensor_" + (i + 1), client)
      }
      // wait for results
      for (i <- 0 until numSensors) {
        results(i) = futures(i).get().value()._2
      }
      // print result
      val line = results.map(t => f"$t%1.3f").mkString("\t| ")
      println(line)
      // wait to send out next queries
      Thread.sleep(refreshInterval)
    }
    client.shutdownAndWait()
  }
  def queryState(
      key: String,
      client: QueryableStateClient)
    : CompletableFuture[ValueState[(String, Double)]] = {
    
    client
      .getKvState[String, ValueState[(String, Double)], (String, Double)](
        JobID.fromHexString(jobId), // jobId
        "maxTemperature", //状态标志符
        key, // 键
        Types.STRING, // 键的类型
        new ValueStateDescriptor[(String, Double)]( //状态描述符
          "", // state name not relevant here 此处与状态名称无关，随便取名
          Types.TUPLE[(String, Double)]))
  }
}
```

