---
title: '[Flink][6][基于时间和窗口的算子]'
date: 2020-11-11 13:56:49
tags:
    - 大数据
    - Flink
    - 流处理
categories:
    - Flink
---
## 第6章 基于时间和窗口的算子





在本章：

1. 首先，我们将学习如何配置**时间特性**、**时间戳**和**水位线**。
2. 然后，我们将介绍**处理函数(process functions)**，它提供了对时间戳和水位线的访问并可以注册计时器，属于比较底层的API。
3. 接下来，我们将使用Flink的**窗口API**，它针对几个最常见的窗口类型都提供了内置实现。
4. 你还将了解如何**自定义窗口算子**。
5. 最后，我们将讨论数据流的**JOIN函数**以及**处理延迟事件**的策略。



### 6.1 配置时间特性



在DataStream API中，您可以使用**时间特性(the time characteristic)**告诉Flink在创建窗口时如何定义时间。时间特性是`StreamExecutionEnvironment`的一个属性，它接受以下值:

| 值                           | 解释                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| **ProcessingTime(处理时间)** | 指定算子根据本地的**机器时钟**来**确定数据流当前的时间**。好处是延迟比较低，坏处是精度很差 |
| **EventTime(事件时间)**      | 指定算子使用来自**数据本身的信息**来确定当前时间。**每个事件**都带有一个**时间戳**，**系统的逻辑时间**由**水位线**定义。 |
| **IngestionTime(摄入时间)**  | 指定每个接收的记录都把**数据源算子的处理时间**作为**事件时间的时间戳**，并**自动生成水位线**。 |



下面举一个设置时间特性的例子

````scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
// 在应用中使用事件时间
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
````



#### 6.1.1 分配时间戳和生成水位线



为了能正常在**事件时间**的时间特性**下**工作，**应用程序**需要向Flink提供两个重要的信息

1. **每个事件**都必须与**时间戳**关联，时间戳通常表示事件实际发生的时间。
2. **事件时间数据流**还需要携带**水位线**，算子从中推断系统的当前事件时间。



**时间戳**和**水位线**都是通过从1970-01-01 00:00:00以来的**毫秒数**指定。**水位线**告诉**算子**，**不必再等**那些**时间戳小于或等于水位线**的事件。



DataStream API中提供了`TimestampAssigner`接口（**时间戳分配器**），用于在**事件**被**输入到流应用后**从事件中**提取时间戳**。通常，时间戳分配器会在数据源生成之后立即调用。此外，为了确保依赖事件时间的算子能正常工作，必须在**任何依赖事件时间**的算子计算**之前**调用**时间戳分配器**



**时间戳分配器**的**工作原理**和其他**转换算子** **类似**。它们会**作用在事件流**上，并**生成**一个**带有时间戳和水位线**的**新数据流**。时间戳分配器**不会更改**DataStream的**数据类型**。



下面展示自定义时间戳分配器的使用方法，主要利用`assignTimestampsAndWatermarks()`方法

````scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val readings: DataStream[SensorReading] = env
	.addSource(new SensorSource)
	// 使用自定义的时间戳分配器来分配时间戳并生成水位线
	.assignTimestampsAndWatermarks(new MyAssigner)
````



自定义时间戳分配器主要分为两种

1. **周期性水位线分配器**：周期性地发出水位线
2. **定点水位线分配器**：根据输入事件中的某个属性或者标记来生成水位线



##### 6.1.1.1 周期性水位线分配器



周期性分配水位线的含义是**系统**以**固定的机器时间间隔**来**发出水位线**并推动事件时钟前进。**默认**的间隔时间设置为**200毫秒**。可以使用`ExecutionConfig.setAutoWatermarkInterval()`方法对**间隔时间**进行**配置**:



```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

//设置水位线间隔为每隔5秒一次
env.getConfig.setAutoWatermarkInternal(5000)
```



示例6-3展示了一个**周期性水位线分配器**，它通过跟踪到目前为止遇到的事件的最大时间戳来生成水位线。当需要生成新水印时，分配器返回一个时间戳等于**最大时间戳减去1分钟容忍间隔**的**水位线**。

````scala
/**定义一个周期性水位线时间戳分配器*/
class PeriodicAssigner extends AssignerWithPeriodicWatermarks[SensorReading] {
    val bound: Long = 60 * 1000 // 1分钟的毫秒数，容忍间隔
    val maxTs: Long = Long.MinValue // 观察到的最大时间戳
    
    /**用来生成水位线的方法*/
    override def getCurrentWatermark: Watermark = {
        new Watermark(maxTs - bound)
    }
    
    /**用来生成时间戳的方法*/
    override def extractTimestamp(
        r: SensorReading
        previousTS: Long): Long = {
        // 更新最大时间戳
        maxTs = maxTs.max(r.timestamp)
        // 返回记录的时间戳
        r.timestamp
    }
}

/**调用这个分配器*/
val readings: DataStream[SensorReading] = env
	.addSource(new SensorSource)
	.assignTimestampAndWatermarks(new PeriodicAssigner())
````



DataStream API内置了**两个**针对**常见情况**的周期性水位线时间戳**分配器**。



###### 1 assignAscendingTimestamps

如果您的输入元素的时间戳是单调递增的，那么您可以使用方法assignascendingtimestamp。此方法使用**当前时间戳**来**生成水位线**。但是这种方式没有考虑延迟情况，因此可能比较**激进**

```scala
val stream: DataStream[SensorReading] = ...
val r = stream.assignAscendingTimestamps(e => e.timestamp)
```



###### 2 BoundedOutOfOrdernessTimestampExtractor



周期性水位线生成的另一种常见情况是，你可以**预测到**在输入流中会遇到的**最大延迟**（任何新到元素的时间戳与所有先前到达的元素的时间戳最大值之间的差异）。对于这种情况，Flink提供了`BoundedOutOfOrdernessTimestampExtractor`，它将**最大预期延迟**作为一个**参数**:



```scala
val stream: DataStream[SensorReading] = ...
val r = stream.assignTimestampAndWatermarks(
    new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Times.seconds(10)){
        override def extractTimestamp(e: SensorReading): Long = e.timestamp
    }
)

"""
我们假设延迟为10秒
并且实现抽象方法extractTimestamp
"""
```



##### 6.1.1.2 定点水位线分配器



有时，输入流包含一些**指示系统进度**的**特殊元组或标记**。Flink为这种情况提供了`assignerwithPunctuatedWatermarks`接口。该接口定义了**`checkAndGetNextWatermark()`方法**，该方法将在**每个事件的`extractTimestamp()`之后**被**调用**。该方法可以**决定是否生成新的水位线**。



下面举个例子

```scala
/**
  * Assigns timestamps to records and emits a watermark for each reading with sensorId == "sensor_1".
  */
class PunctuatedAssigner extends AssignerWithPunctuatedWatermarks[SensorReading] {

  // 1 min in ms
  val bound: Long = 60 * 1000

  // 如果该方法返回一个非空、且大于之前值的水位线，算子会将这个新水位线发出
  override def checkAndGetNextWatermark(r: SensorReading, extractedTS: Long): Watermark = {
    if (r.id == "sensor_1") {
      // emit watermark if reading is from sensor_1
      new Watermark(extractedTS - bound)
    } else {
      // do not emit a watermark
      null
    }
  }
  // 生成时间戳	
  override def extractTimestamp(r: SensorReading, previousTS: Long): Long = {
    // assign record timestamp
    r.timestamp
  }
}
```



#### 6.1.2 水位线、延迟及完整性问题



水位线主要用于平衡**延迟**和**结果的完整性**。

- 如果水位线设置的过于**宽松**，会导致**较大的延迟**并且需要**更多的存储空间**来缓存数据；但是得到的**结果相对准确**
- 如果水位线设置的过于**紧迫**，则**相反**
- 但是无论水位线设置得多宽松，**总会出现迟到数据**
- 对于流处理应用，就是要在延迟和完整性之间做一个**取舍**



### 6.2 处理函数(Process Function)



下面看看如何在**转换算子**中**访问**到**时间戳和水位线**信息。



DataStream API提供一系列相对底层的转换操作——**处理函数**，这些转换的语义很强大，

- 可以**访问**事件的**时间戳**和**水位线**，
- 还可以**注册**在特定时间触发的**计时器**，
- 还可以通过**副输出(side output)**功能，发出记录到多个输出流。



目前，Flink提供了**8种**不同的**处理函数**:

- ProcessFunction
- KeyedProcessFunction
- CoProcessFunction
- ProcessJoinFunction
- BroadcastProcessFunction
- KeyedBroadcastProcessFunction
- ProcessWindowFunction
- ProcessAllWindowFunction。



下面以KeyedProcessFunction为例

- KeyedProcessFunction是一个**非常灵活**的函数，**作用于KeyedStream**上。
- 对流的**每条记录**调用该函数，会返回零条、一条或多条记录。
- 并且它**实现了RichFunction接口**，因此提供了open()、close()和getRuntimeContext()方法。
- 另外，KeyedProcessFunction[KEY,  IN, OUT]还额外提供了以下两种抽象方法:
  1. processElement(v: IN, ctx: Context, out:  Collector[out])：**Context**是进程函数的**灵活之处**。它提供对**当前事件**的**时间戳**、**键**、**TimerService**等的**访问**。此外，Context可以将记录发送到副输出。
  2. onTimer(timestamp:Long, ctx: OnTimerContext, out:  Collector[out])：它是一个回调函数，当之前注册的**计时器触发时**，它会被调用，来**执行**计时器绑定的**逻辑操作**。



KeyedProcessFunction接口的源代码如下

```java
/**
 * A keyed function that processes elements of a stream.
 *
 * @param <K> Type of the key.
 * @param <I> Type of the input elements.
 * @param <O> Type of the output elements.
 */
@PublicEvolving
public abstract class KeyedProcessFunction<K, I, O> extends AbstractRichFunction {

	private static final long serialVersionUID = 1L;

	public abstract void processElement(
        I value, Context ctx, Collector<O> out) throws Exception;

	public void onTimer(
        long timestamp, OnTimerContext ctx, Collector<O> out) throws Exception {}

	/**
	 * Information available in an invocation of {@link #processElement(Object, Context, Collector)}
	 * or {@link #onTimer(long, OnTimerContext, Collector)}.
	 */
	public abstract class Context {

		public abstract Long timestamp();
        
		public abstract TimerService timerService();

		public abstract <X> void output(OutputTag<X> outputTag, X value);

		public abstract K getCurrentKey();
	}

	/**
	 * Information available in an invocation of {@link #onTimer(long, OnTimerContext, Collector)}.
	 */
	public abstract class OnTimerContext extends Context {

		public abstract TimeDomain timeDomain();

		@Override
		public abstract K getCurrentKey();
	}

}
```



#### 6.2.1 TimerService和Timer



观察上面的源码我们不难发现，`Context`提供一个`timerSercvice()`方法，它会返回一个`TimerService`。这个接口提供了一系列时间相关的操作。具体如下源码



````java
public interface TimerService {

    /**返回当前的处理时间*/
	long currentProcessingTime();

    /**返回当前的水位线*/
	long currentWatermark();

    /**注册处理时间计时器*/
	void registerProcessingTimeTimer(long time);

    /**注册事件时间计时器*/
	void registerEventTimeTimer(long time);

    /**移除处理时间计时器*/
	void deleteProcessingTimeTimer(long time);

    /**移除事件时间计时器*/
	void deleteEventTimeTimer(long time);
}
````



在KeyedProcessFunction中，**每个键**的**每个时间戳**只能注册**一个计时器**，这些计时器会按照时间戳顺序放到一个**优先队列**中。



当需要生成检查点时，**计时器也会被写入检查点**。如果应用程序需要**从故障中恢复**，那么在应用程序重新启动时**过期的所有处理时间计时器**将在应用程序恢复时**立即触发**。



下面我们实现一个KeyedProcessFunction。它监测传感器的温度，如果传感器的温度在处理时间语义中单调增加1秒，就会发出警告:

````scala
object ProcessFunctionTimers {

  def main(args: Array[String]) {

    // set up the streaming execution environment
    val env = StreamExecutionEnvironment.getExecutionEnvironment

    // use event time for the application
    env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)

    // ingest sensor stream
    val readings: DataStream[SensorReading] = env
      // SensorSource generates random temperature readings
      .addSource(new SensorSource)

    val warnings = readings
      // key by sensor id
      .keyBy(_.id)
      // 调用KeyedProcessFunction的具体实现TempIncreaseAlertFunction
      .process(new TempIncreaseAlertFunction)

    warnings.print()

    env.execute("Monitor sensor temperatures.")
  }
}

/** Emits a warning if the temperature of a sensor
  * monotonically increases for 1 second (in processing time).
  */
class TempIncreaseAlertFunction
  extends KeyedProcessFunction[String, SensorReading, String] {

  // 存储上一个到来的传感器温度
  lazy val lastTemp: ValueState[Double] =
    getRuntimeContext.getState(
      new ValueStateDescriptor[Double]("lastTemp", Types.of[Double])
    )

  // 存储当前活跃的一个计时器
  lazy val currentTimer: ValueState[Long] =
    getRuntimeContext.getState(
      new ValueStateDescriptor[Long]("timer", Types.of[Long])
    )
  /**处理每个元素*/
  override def processElement(
      r: SensorReading,
      ctx: KeyedProcessFunction[String, SensorReading, String]#Context,
      out: Collector[String]): Unit = {

    // get previous temperature
    val prevTemp = lastTemp.value()
    // update last temperature
    lastTemp.update(r.temperature)

    val curTimerTimestamp = currentTimer.value()
    // 这个计时器的第一个事件到来了，不做任何处理  
    if (prevTemp == 0.0) {
      // first sensor reading for this key.
      // we cannot compare it with a previous value.
    }
    // 如果新的温度小于老的温度，在上下文中注销计时器并且清空保存的计时器状态  
    else if (r.temperature < prevTemp) {
      // temperature decreased. Delete current timer.
      ctx.timerService().deleteProcessingTimeTimer(curTimerTimestamp)
      currentTimer.clear()
    }
    // 如果新的温度大于老的温度并且当前没有计时器，就创建一个1s后触发的计时器并且保存到内部状态里面  
    else if (r.temperature > prevTemp && curTimerTimestamp == 0) {
      // temperature increased and we have not set a timer yet.
      // set timer for now + 1 second
      val timerTs = ctx.timerService().currentProcessingTime() + 1000
      ctx.timerService().registerProcessingTimeTimer(timerTs)
      // remember current timer
      currentTimer.update(timerTs)
    }
  }
      
  /**当计时器触发时会调用这个函数*/		
  override def onTimer(
      ts: Long,
      ctx: KeyedProcessFunction[String, SensorReading, String]#OnTimerContext,
      out: Collector[String]): Unit = {
    // 把这个事件的String放到输出中，相当于发出了一条警报 
    out.collect("Temperature of sensor '" + ctx.getCurrentKey +
      "' monotonically increased for 1 second.")
    // 清空计时器状态
    currentTimer.clear()
  }
}
````



#### 6.2.2 向副输出发送数据（Emitting to Side Outputs）



**副输出(side outputs)**是处理函数的一个特性，它可以从同一个函数**发出多条数据流**，且**副输出**的元素**类型**可以与输入**不同**。



下面直接举个例子

```scala
/** 
 * 对于温度低于32F的读数，会向副输出发出警报
 */
object SideOutputs {

  def main(args: Array[String]): Unit = {

    // ingest sensor stream
    val readings: DataStream[SensorReading] = ...

    val monitoredReadings: DataStream[SensorReading] = readings
      // monitor stream for readings with freezing temperatures
      // 调用处理函数
      .process(new FreezingMonitor)

    // retrieve and print the freezing alarms
    monitoredReadings
      .getSideOutput(new OutputTag[String]("freezing-alarms"))
      .print()

    // print the main output
    readings.print()

    env.execute()
  }
}

/** Emits freezing alarms to a side output for readings with a temperature below 32F. */
class FreezingMonitor extends ProcessFunction[SensorReading, SensorReading] {

  // define a side output tag
  // 定义一个副输出标签  
  lazy val freezingAlarmOutput: OutputTag[String] =
    new OutputTag[String]("freezing-alarms")
  
  // 处理每个元素的方法  
  override def processElement(
      r: SensorReading,
      ctx: ProcessFunction[SensorReading, SensorReading]#Context,
      out: Collector[SensorReading]): Unit = {
    // 如果SensorReading的温度小于32F则放入Freezing Alarm副输出  
    // emit freezing alarm if temperature is below 32F.
    if (r.temperature < 32.0) {
      ctx.output(freezingAlarmOutput, s"Freezing Alarm for ${r.id}")
    }
    // 所有的SensorReading都output到常规输出  
    // forward all readings to the regular output
    out.collect(r)
  }
}

```



#### 6.2.3 CoProcessFunction



对于有两个输入的底层操作，DataStream API还提供了CoProcessFunction。与CoFlatMapFunction类似，CoProcessFunction也提供了一对作用在每个输入上的转换方法`processElement1()`和`processElement2()`。



下面直接举个例子

```scala
object CoProcessFunctionTimers {

  def main(args: Array[String]) {

    // set up the streaming execution environment
    val env = StreamExecutionEnvironment.getExecutionEnvironment

    // use event time for the application
    env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)

    // switch messages disable filtering of sensor readings for a specific amount of time
    val filterSwitches: DataStream[(String, Long)] = env
      .fromCollection(Seq(
        ("sensor_2", 10 * 1000L), // sensor_2 前进2秒
        ("sensor_7", 60 * 1000L)) // sensor_7 前进7秒
      )

    // ingest sensor stream
    val readings: DataStream[SensorReading] = env
      // SensorSource generates random temperature readings
      .addSource(new SensorSource)

    val forwardedReadings = readings
      // 联结读数和开关 connect readings and switches 
      .connect(filterSwitches)
      // 设定这两个流分别根据什么来分区 key by sensor ids 
      .keyBy(_.id, _._1)
      // 调用下面代码实现的一个CoProcessFunction的接口 apply filtering CoProcessFunction 
      .process(new ReadingFilter)

    forwardedReadings
      .print()

    env.execute("Monitor sensor temperatures.")
  }
}

class ReadingFilter
  extends CoProcessFunction[SensorReading, (String, Long), SensorReading] {

  // 一个Boolean类型的状态 允许转发的开关 switch to enable forwarding
  lazy val forwardingEnabled: ValueState[Boolean] =
    getRuntimeContext.getState(
      new ValueStateDescriptor[Boolean]("filterSwitch", Types.of[Boolean])
    )

  // 保存一个时钟 hold timestamp of currently active disable timer
  lazy val disableTimer: ValueState[Long] =
    getRuntimeContext.getState(
      new ValueStateDescriptor[Long]("timer", Types.of[Long])
    )
  
  // 处理第一条流的数据    
  override def processElement1(
      reading: SensorReading,
      ctx: CoProcessFunction[SensorReading, (String, Long), SensorReading]#Context,
      out: Collector[SensorReading]): Unit = {
    // 检查开关是否为true，true的时候才转发，false时直接丢弃应该是     
    // check if we may forward the reading
    if (forwardingEnabled.value()) {
      out.collect(reading)
    }
  }
  
  // 处理第二条流的数据    
  override def processElement2(
      switch: (String, Long),
      ctx: CoProcessFunction[SensorReading, (String, Long), SensorReading]#Context,
      out: Collector[SensorReading]): Unit = {
    
    // 打开转发开关
    // enable reading forwarding
    forwardingEnabled.update(true)
    // 设置停止转发的计时器  
    // set disable forward timer
    val timerTimestamp = ctx.timerService().currentProcessingTime() + switch._2
    val curTimerTimestamp = disableTimer.value()
    // 比较和当前的计时器相比哪个比较大，如果新计时器大就移除老计时器，重新设置为新计时器
    if (timerTimestamp > curTimerTimestamp) {
      // remove current timer and register new timer
      ctx.timerService().deleteProcessingTimeTimer(curTimerTimestamp)
      ctx.timerService().registerProcessingTimeTimer(timerTimestamp)
      disableTimer.update(timerTimestamp)
    }
  }
  
  // 计时器被触发时，这个方法会被调用    
  override def onTimer(
      ts: Long,
      ctx: CoProcessFunction[SensorReading, (String, Long), SensorReading]#OnTimerContext,
      out: Collector[SensorReading]): Unit = {

    // remove all state. Forward switch will be false by default.
    // 开关会被设置为false，也就是停止转发
    forwardingEnabled.clear()
    disableTimer.clear()
  }
}
```



### 6.3 窗口算子



**窗口**是流式应用中的常见操作。它们可以在**无限数据流**的**有界区间**上实现**聚合**等操作。通常，这些间隔是使用基于时间的逻辑定义的。窗口算子提供了一种基于有限大小的桶对事件进行分组的方法，并对这些桶中的有限内容进行计算。



#### 6.3.1 定义窗口算子



窗口算子可以应用于键值分区(keyed)或非键值分区(nonkeyed)的数据流上。**键值分区**上的窗口算子**并行计算**，而**非键值分区**的窗口算子在**单个线程**中处理。



在流上应用窗口算子需要两步：

1. 第一步是调用`keyBy()`指定一个**窗口分配器**，它会决定对输入流中的元素如何划分到各个桶中
2. 第二步是调用某一种**窗口函数**来**处理**分配到窗口中的元素



下面的例子展示了分区和不分区的**窗口算子定义方式**



````scala
stream
	.keyBy(...)		// 分区
	.window(...)	// 指定窗口分配器
	.reduce/aggregate/process(...)	// 指定窗口函数
````



````scala
stream
	.windowAll(...)	//指定窗口分配器，不分区(window-all，全量窗口)
	.reduce/aggregate/process(...)	// 指定窗口函数
````



#### 6.3.2 内置窗口分配器(Built-in Window Assigners)



Flink为最常见的窗口使用场景提供了内置的窗口分配器。本节我们**只讨论基于时间**的窗口分配器。基于时间的窗口分配器**根据**元素**事件时间**的时间戳或当前**处理时间**将元素**分配**给窗口。**每个时间窗口**有一个**开始时间戳**和一个**结束时间戳**。



所有**内置的窗口分配器**都提供了一个**默认触发器**，当(处理或事件)时间经过**窗口末尾**时，该触发器将触发对窗口的计算，然后指定的**窗口函数**就会**被调用**。



Flink的内置窗口分配器**所创建的窗口**的**类型**为`TimeWindow`。此**窗口类型**实际上表示两个时间戳之间的**时间区间**（左闭右开）。



##### 6.3.2.1 滚动窗口(Tumbling windows)



滚动窗口分配器将元素放入**不重叠**的**固定大小**的窗口中，如图6-1所示。

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201110183732.png)



Datastream API提供了两个分配器：

- TumblingEventTimeWindows：用于**事件时间**
- TumblingProcessingTimeWindow：用于**处理时间**
- 滚动窗口分配器只接收一个参数：**窗口大小**



例子如下

```scala
val sensorData: DataStream[SensorReading] = ...

// 按照事件时间来分配窗口
val avgTemp = sensorData
  .keyBy(_.id)
   // 按照事件时间，大小为1s，来划分窗口
  .window(TumblingEventTimeWindows.of(Time.seconds(1)))
  .process(new TemperatureAverager)

// 按照处理时间来分配窗口
val avgTemp = sensorData
  .keyBy(_.id)
   // 按照处理时间，大小为1s，来划分窗口
  .window(TumblingProcessingTimeWindows.of(Time.seconds(1)))
  .process(new TemperatureAverager)

// 利用一个更快捷的方法来分配窗口
val avgTemp = sensorData
  .keyBy(_.id)
   // 大小为1s，来划分窗口,具体按照处理时间还是事件时间要结合当前环境来进行判断
  .timeWindow(Time.second(1))
  .process(new TemperatureAverager)
```



默认情况下，滚动窗口与纪元时间1970-01-01-00:00:00.000对齐。例如，大小为1小时的分配器将在00:00:00、01:00:00、02:00:00等位置定义窗口。或者，你可以在分配器中通过第二个参数指定一个**偏移量。**下面的代码显示了偏移值为15分钟的窗口，它从00:15:00、01:15:00、02:15:00等位置定义窗口：



````scala
val avgTemp = sensorData
  .keyBy(_.id)
   // group readings in 1 hour windows with 15 min offset
  .window(TumblingEventTimeWindows.of(Time.hours(1), 
Time.minutes(15)))
  .process(new TemperatureAverager)
````



##### 6.3.2.2 滑动窗口(Sliding windows)



滑动窗口分配器将元素分配给**大小固定**且**按指定滑动间隔移动**的窗口，如图6-2所示

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201110185009.png)



对于滑动窗口，必须指定**窗口大小**和**滑动间隔**，以定义新窗口启动的频率。

- 当**滑动间隔小于窗口大小**时，**窗口会重叠**，元素可以分配给多个窗口。
- 当**滑动间隔大于窗口大小**时，有些元素不会被分配给任何窗口，会被**丢弃**。



下面举个例子

````scala
// 事件时间
// event-time sliding windows assigner
val slidingAvgTemp = sensorData
  .keyBy(_.id)
   // create 1h event-time windows every 15 minutes
  .window(SlidingEventTimeWindows.of(Time.hours(1), Time.minutes(15)))
  .process(new TemperatureAverager)

// 处理时间
// processing-time sliding windows assigner
val slidingAvgTemp = sensorData
  .keyBy(_.id)
   // create 1h processing-time windows every 15 minutes
  .window(SlidingProcessingTimeWindows.of(Time.hours(1), Time.minutes(15)))
  .process(new TemperatureAverager)

// 简写
// sliding windows assigner using a shortcut method
val slidingAvgTemp = sensorData
  .keyBy(_.id)
   // shortcut for window.(SlidingEventTimeWindow.of(size, slide))
  .timeWindow(Time.hours(1), Time(minutes(15)))
  .process(new TemperatureAverager)
````



##### 6.3.2.3 会话窗口(Session windows)



会话窗口分配器将元素放入**长度可变**但是**不重叠**的窗口中。**会话窗口的边界**由**不活动时间间隔**(session gap)（没有接收到记录的时间间隔）定义。



图6-3说明了如何将元素分配给会话窗口。

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201110192139.png)



下面举一个例子

````scala
/** session gap 设置为15分钟*/
// event-time session windows assigner
val sessionWindows = sensorData
  .keyBy(_.id)
   // create event-time session windows with a 15 min gap
  .window(EventTimeSessionWindows.withGap(Time.minutes(15)))
  .process(...)

// processing-time session windows assigner
val sessionWindows = sensorData
  .keyBy(_.id)
   // create processing-time session windows with a 15 min gap
  .window(ProcessingTimeSessionWindows.withGap(Time.minutes(15)))
  .process(...)
````



由于会话窗口的**开始时间**和**结束时间**都**依赖于输入元素**，所以窗口分配器**不能立即**将所有元素**分配**给**正确**的窗口。

- 因此，会话窗口分配器**最初**将**每个输入元素**映射到自己的**单独窗口**中，**开始时间**为元素的**时间戳**，**窗口大小**为**会话间隔**。
- 然后，**分配器**会将**具有重叠范围**的所有窗口**合并**。



#### 6.3.3 在窗口上应用函数



如6.3.1小节所示，**窗口的计算逻辑**由**窗口函数**负责定义。



可用于窗口的函数类型有两种

1. **增量聚合函数**（Incremental aggregation functions）：
   - 它的应用场景是**窗口**内**以状态形式** **存储某个值**并且需要**根据每个加入窗口的元素**对该值进行**更新**
   - 此类函数通常非常**节省空间**且最终会将**聚合值**作为单个结果发出
   - 下文介绍的**ReduceFunction**和**AggregateFunction**都是增量聚合函数
2. **全量窗口函数**（Full window functions）
   - 全量窗口函数**收集**一个**窗口的所有元素**，并在**计算**时**遍历所有元素**来获取计算结果。
   - 全窗口函数通常**需要更多空间**，但比增量聚合函数**支持更复杂的逻辑**。
   - 下文介绍的**ProcessWindowFunction**就是一个全量窗口函数。



##### 6.3.3.1 ReduceFunction



**ReduceFunction接受两个相同类型的值，并将它们组合成一个类型相同的值。**当在一个Windowed Stream上应用ReduceFunction语义时，ReduceFunction增量地聚合窗口中的元素。窗口只存储聚合的**当前结果**，它是一个**和输入输出类型相同的值**。当接收到新元素时，算子会从窗口读取当前状态并调用ReduceFunction结合新元素来更新状态。



下面举个例子

```scala
val minTempPerWindow: DataStream[(String, Double)] = sensorData
  .map(r => (r.id, r.temperature))
  .keyBy(_._1)
  .timeWindow(Time.seconds(15))
  // 计算并输出15s窗口中的最小值 	
  .reduce((r1, r2) => (r1._1, r1._2.min(r2._2)))
```



##### 6.3.3.2 AggregateFunction



AggregateFunction是一个比ReduceFunction**更灵活**的窗口函数，其接口定义如下

```scala
/**
* IN 输入类型
* ACC 累加器类型(内部状态)
* OUT 输出类型
*/
public interface AggregateFunction<IN, ACC, OUT> extends Function, Serializable {
  // 创建一个累加器来启动聚合
  ACC createAccumulator();
  // 向累加器中添加一个输入元素并返回累加器
  ACC add(IN value, ACC accumulator);
  // 根据累加器来返回结果
  OUT getResult(ACC accumulator);
  // 合并两个累加器
  ACC merge(ACC a, ACC b);
}
```

与ReduceFunction不同的是，AggregateFunction的**中间数据类型**和**输出类型** **不依赖**于**输入类型**。



下面举个例子

```scala
// 计算每个窗口的传感器读数的平均温度。
val avgTempPerWindow: DataStream[(String, Double)] = sensorData
  .map(r => (r.id, r.temperature))
  // 根据id来分key	
  .keyBy(_._1)
  // 15秒的窗口	
  .timeWindow(Time.seconds(15))
  .aggregate(new AvgTempFunction)

// An AggregateFunction to compute the average tempeature per sensor.
// The accumulator holds the sum of temperatures and an event count.
class AvgTempFunction extends AggregateFunction 
[(String, Double), (String, Double, Int), (String, Double)] {
  
  // 创建累加器，累加器的  
  override def createAccumulator() = {("", 0.0, 0) // (ID, 累加器, 计数器}
  
  // 加和  
  override def add(in: (String, Double), acc: (String, Double, Int)) = {
    (in._1, in._2 + acc._2, 1 + acc._3)
  }
  
  // 求平均作为结果返回                                    
  override def getResult(acc: (String, Double, Int)) = {
    (acc._1, acc._2 / acc._3)
  }
  
  // 合并累加器的方法                                    
  override def merge(acc1: (String, Double, Int), acc2: (String, Double, Int)) = {
    (acc1._1, acc1._2 + acc2._2, acc1._3 + acc2._3)
  }
}
```



##### 6.3.3.3 ProcessWindowFunction



ProcessWindowFunction是一个Full Window Function，它会将窗口的所有元素**收集**起来**先不做处理**，等**完全收集好**之后，**再处理**。它比增量聚合应用更广，比如计算窗口内数据的中值或者出现频率最高的值等。



````java
/**
* IN: 输入类型
* OUT: 输出类型
* KEY: 键的类型
* W: 窗口元数据的类型
*/
public abstract class ProcessWindowFunction<IN, OUT, KEY, W extends Window> 
    extends AbstractRichFunction {
  
  // 对窗口执行计算
  void process(KEY key, Context ctx, Iterable<IN> vals, 
               Collector<OUT> out) throws Exception;
  
  // 当窗口要被删除时，清理一些自定义的状态
  public void clear(Context ctx) throws Exception {}
  
  // Context窗口的上下文
  public abstract class Context implements Serializable {
      
    // 返回窗口元数据
    public abstract W window();

    // 返回当前处理时间
    public abstract long currentProcessingTime();

    // 返回当前事件时间戳
    public abstract long currentWatermark();

    // State accessor for per-window state 每个窗口的状态
    public abstract KeyedStateStore windowState();

    // State accessor for per-key global state 每个键的全局状态
    public abstract KeyedStateStore globalState();

    // Emits a record to the side output identified by the OutputTag.
    // 向OutputTag标识的副输出发送记录  
    public abstract <X> void output(OutputTag<X> outputTag, X value);
  }
}
````



`process()`和`clear()`都有一个Context对象作为参数，这个参数功能很强大

- 访问窗口元数据(Window类型)
- 访问当前处理时间和水位线
- 管理每个窗口的状态和每个键的全局状态
  - **窗口状态**只有**当前窗口**才能访问到
  - **全局状态**可用于在**同一键上的多个窗口**之间共享信息。
- 副输出



注意：**使用了窗口状态**这一功能的ProcessWindowFunction**需要实现clear()**方法，来**在窗口被删除之前清理自定义的窗口状态**。



下面举个例子

````scala
// output the lowest and highest temperature reading every 5 seconds
val minMaxTempPerWindow: DataStream[MinMaxTemp] = sensorData
  // 这里keyBy中key的类型需要与ProcessWindowFunction的KEY类型参数一致
  .keyBy(_.id)
  .timeWindow(Time.seconds(5))
  .process(new HighAndLowTempProcessFunction)

case class MinMaxTemp(id: String, min: Double, max:Double, endTs: Long)

/**
 * A ProcessWindowFunction that computes the lowest and highest 
temperature
 * reading per window and emits them together with the 
 * end timestamp of the window.
 * [IN, OUT, KEY, W]
 */
class HighAndLowTempProcessFunction extends ProcessWindowFunction
[SensorReading, MinMaxTemp, String, TimeWindow] {
  
    override def process(key: String, // 键：这里是传感器的ID 
                         ctx: Context,// 上下文
      					 vals: Iterable[SensorReading],// 桶中的全部元素 
                         out: Collector[MinMaxTemp]): Unit = { // 输出
      
      val temps = vals.map(_.temperature)
      val windowEnd = ctx.window.getEnd
      out.collect(MinMaxTemp(key, temps.min, temps.max, windowEnd))
  
  }
}
````



##### 6.3.3.4 增量聚合与ProcessWindowFunction结合使用



很多情况下用于窗口上的**逻辑**都可以**表示为增量聚合**，只不过它还**需要访问窗口的元数据或状态**。可以将增量聚合函数与ProcessWindowFunction结合使用。

- **分配给窗口的元素**将**立即聚合**，
- 当窗口的触发器触发时，**聚合的结果**将被**传递给ProcessWindowFunction**
- process()方法的`Iterable`参数将**只提供单个值**，即增量聚合的结果。



在DataStream API中，这实现上述过程的途径是**将ProcessWindowFunction作为reduce()或aggregate()方法的第二个参数**，如下面的代码所示:

```scala
input
  .keyBy(...)
  .timeWindow(...)
  .reduce(
    incrAggregator: ReduceFunction[IN],
    function: ProcessWindowFunction[IN, OUT, K, W])
input
  .keyBy(...)
  .timeWindow(...)
  .aggregate(
    incrAggregator: AggregateFunction[IN, ACC, V],
    windowFunction: ProcessWindowFunction[V, OUT, K, W])
```



下面举一个例子

```scala
case class MinMaxTemp(id: String, min: Double, max:Double, endTs: Long)

val minMaxTempPerWindow2: DataStream[MinMaxTemp] = sensorData
  .map(r => (r.id, r.temperature, r.temperature))
  .keyBy(_._1)
  .timeWindow(Time.seconds(5))
  .reduce(
    
    // 增量计算最低和最高温度 [IN, ACC, V] 
    (r1: (String, Double, Double), r2: (String, Double, Double)) => {
      (r1._1, r1._2.min(r2._2), r1._3.max(r2._3))
    },
    
    // 在ProcessWindowFunction中计算最终结果  
    new AssignWindowEndProcessFunction()
  )

// [V, OUT, K, W]
class AssignWindowEndProcessFunction extends ProcessWindowFunction
[(String, Double, Double), MinMaxTemp, String, TimeWindow] {
  
  override def process(
      key: String,
      ctx: Context,
      minMaxIt: Iterable[(String, Double, Double)],
      out: Collector[MinMaxTemp]): Unit = {
    
    // 取下序列中的唯一一个元素
    val minMax = minMaxIt.head
    // 从上下文中获取窗口的结束时间  
    val windowEnd = ctx.window.getEnd
    // 输出  
    out.collect(MinMaxTemp(key, minMax._2, minMax._3, windowEnd))
  
  }
}
```



#### 6.3.4 自定义窗口算子



基于Flink的**内置窗口分配器** **定义的窗口算子**可以**应对许多常见情况**。但是如果你需要更复杂的逻辑，也可以自定义窗口算子。DataStream API**对外暴露**了**自定义窗口算子**的接口和方法。你可以实现自己的**分配器(assigner)**、**触发器(trigger)**和**移除器(evictor)**，再加上前面一节提到的**窗口函数**，就可以**组合出一个自定义窗口算子**。



当一个**元素到达**一个窗口算子时，它将**被传递给窗口分配器**。该**分配器**会**决定** **元素需要被放置在哪个窗口**。如果这个窗口还不存在，就会直接创建。



如果窗口算子**配置了增量聚合函数**，则会**立即聚合**新添加的元素，并**将结果存储为窗口的状态**。如果窗口算子**没有配置增量聚合函数**，则将新元素**追加到**一个用来存储所有窗口分配元素的**ListState**上。



每当一个元素被添加到一个窗口时，它也被传递到该窗口的**触发器**。触发器定义**何时执行窗口计算**、**何时清除窗口及其状态**。



**触发器成功触发后会调用窗口函数**，根据窗口函数的不同，触发器的行为具体来说分以下三种

| 说明                                                         | 图例                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 如果算子**只配置了增量聚合函数**，则调用`getResult()`发出当前聚合结果。 | ![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201110211150.png) |
| 如果算子只配置了`ProcessWindowFunction`（**全量窗口函数**），那么该函数的`process()`将被调用并发出结果 | ![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201110211603.png) |
| 如果算子**既配置了增量聚合函数，又配置全量窗口函数**，则对聚合函数的**聚合值** **应用全量窗口函数**并发出结果。（具体见6.3.3.4小节） | ![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201110211833.png) |



**移除器**是一个**可选组件**，可以在调用ProcessWindowFunction**之前**或**之后**注入。移除器可以**从窗口中删除**所收集的**某些元素**。因为它必须遍历所有元素，所以只能在没有指定增量聚合函数的情况下使用它（指定了增量聚合函数的话，此时已经增量聚合结束了，再删除元素会导致计算结果不准确）。



下面的代码展示了如何**组合**前文介绍的各种组件，来**生成自定义算子**

````scala
stream
  .keyBy(...)
  .window(...)                   // specify the window assigner 指定分配器
 [.trigger(...)]                 // optional: specify the trigger 指定触发器
 [.evictor(...)]                 // optional: specify the evictor 指定移除器
  .reduce/aggregate/process(...) // specify the window function 指定窗口函数
````



此外，当没有显式指定`trigger`时，Flink会提供一个默认的`trigger`，它会在时间戳移动到**窗口右边界**时**触发**



##### 6.3.4.1 窗口的生命周期



在本节中，我们将讨论窗口的生命周期——何时创建，由哪些信息组成，何时删除。



###### 6.3.4.1.1 何时创建



当**窗口分配器**需要**向窗口分配第一个元素**时，就会**创建**一个窗口。因此，一个窗口至少包含一个元素。



###### 6.3.4.1.2 由哪些信息组成



一个窗口由以下不同的状态组成:

| 状态                     | 描述                                                         |
| ------------------------ | ------------------------------------------------------------ |
| **窗口内容**             | 如果窗口算子配置了ReduceFunction或AggregateFunction，则窗口内容包含**增量聚合的结果**。如果窗口算子配置了ProcessFunction，则窗口内容包含**分配给窗口的元素** |
| **窗口对象**             | 窗口分配器返回零个、一个或多个窗口对象。窗口算子根据返回的对象对元素进行分组。因此**窗口对象中保存**用于**区分窗口的信息**。每个窗口对象都有一个**结束时间戳**，它定义了可以删除窗口及其状态的时间点。 |
| **触发器定时器**         | 可以在触发器中注册计时器，以便在特定的时间点回调             |
| **触发器中的自定义状态** | 触发器可以定义和使用针对每个窗口、每个键的**自定义状态**。这种状态完全**由触发器控制**，而不是由窗口算子维护。 |



###### 6.3.4.1.3 何时删除



窗口算子会在**窗口结束时间**(由窗口对象的结束时间戳定义)**删除窗口**。



删除一个窗口时，**窗口算子**会**自动清除** **窗口内容**并丢弃**窗口对象**，但**不会清除** **自定义的触发器状态**和**触发器计时器**。因此，**触发器**必须**实现`trigger.clear()`方法**来**清理**这些，以**防止状态泄漏**。



##### 6.3.4.2 窗口分配器



WindowAssigner用于决定将到达的元素分配给哪些窗口。



下面首先看看`WindowAssigner`接口的源码

```java
// T: 流中的元素类型
// W: 窗口元数据类型
public abstract class WindowAssigner<T, W extends Window> implements Serializable {
  
  // Returns a collection of windows to which the element is assigned
  // 返回元素分配的目标窗口集合  
  public abstract Collection<W> assignWindows(T element, 
                                              long timestamp,
                                              WindowAssignerContext context);
  
  // 返回窗口分配器的默认触发器(用于算子没有显式指定触发器的情况)
  public abstract Trigger<T, W> getDefaultTrigger(StreamExecutionEnvironment env);
  
  // Returns the TypeSerializer for the windows of this WindowAssigner
  public abstract TypeSerializer<W> getWindowSerializer(ExecutionConfig executionConfig);
  
  // 判断这个窗口分配器使用的是不是事件时间
  public abstract boolean isEventTime();
  
  // 窗口分配器的上下文
  public abstract static class WindowAssignerContext {
    
  	// 返回当前处理时间
  	public abstract long getCurrentProcessingTime();
  }
}
```



下面展示如何**自定义**一个**窗口分配器**

```scala
/** A custom window that groups events into 30-second tumbling windows. */
class ThirtySecondsWindows extends WindowAssigner[Object, TimeWindow]
  // 窗口尺寸  
  val windowSize: Long = 30 * 1000L
  
  // 分配窗口  
  override def assignWindows(o: Object,ts: Long,
  	ctx: WindowAssigner.WindowAssignerContext):java.util.List[TimeWindow] = {
    
    // 计算所属窗口的开始时间和结束时间
    // rounding down by 30 seconds
    val startTime = ts - (ts % windowSize)
    val endTime = startTime + windowSize
    // 返回一个列表，列表中的每个元素都是当前事件所属的窗口  
    // emitting the corresponding time window
    Collections.singletonList(new TimeWindow(startTime, endTime))
  }
  
  // 获取默认触发器	
  override def getDefaultTrigger(env: environment.StreamExecutionEnvironment)
    : Trigger[Object, TimeWindow] = {
     // 直接返回一个事件时间触发器
     EventTimeTrigger.create()
  }

  // 窗口序列化器    
  override def getWindowSerializer(executionConfig: ExecutionConfig)
    :TypeSerializer[TimeWindow] = {
    new TimeWindow.Serializer
  }
  // 使用的是处理时间  
  override def isEventTime = true
}
```



##### 6.3.4.3 触发器



**触发器**定义了**何时**进行**窗口计算**并**发出结果**。触发器可以根据**时间**或**特定的数据条件**触发。例如对于基于时间窗口的默认触发器来说，当处理时间或水位线超过窗口结束边界的时间戳时，默认触发器将触发。



触发器的功能很强大，它可以访问时间属性和计时器，并且可以使用状态。



每次**调用触发器**时，它都会**生成一个TriggerResult**来决定窗口应该发生什么。TriggerResult可以取以下值之一:

| TriggerResult | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| CONTINUE      | 什么都不做                                                   |
| FIRE          | 如果窗口算子配置了**ProcessWindowFunction**，则调用该函数**计算并发出**结果。如果窗口只配置了**增量聚合函数**，则会**发出**当前聚合结果。 |
| PURGE         | **窗口内容**将被完全**丢弃**，**窗口**将被**删除**。此外，**ProcessWindowFunction.clear()**方法被调用以清除所有自定义的每个窗口状态。 |
| FIRE_AND_PURG | 首先**计算**窗口(触发)，然后**删除**所有状态和元数据(清除)。 |



下面展示一下触发器的接口源码

````java
public abstract class Trigger<T, W extends Window> implements Serializable {
    
  // 每当有元素被添加到窗口时，这个方法都会被调用  
  // Called for every element that gets added to a window
  TriggerResult onElement(T element, long timestamp, W window, TriggerContext ctx);
  
  // 当一个处理时间计时器触发时调用  
  // Called when a processing-time timer fires
  public abstract TriggerResult onProcessingTime(long timestamp,
                                                 W window, TriggerContext ctx);
  // 当一个事件时间计时器触发时调用  
  // Called when an event-time timer fires
  public abstract TriggerResult onEventTime(long timestamp, 
                                            W window, TriggerContext ctx);
    
  // 返回触发器是否支持状态的合并  
  // Returns true if this trigger supports merging of trigger state
  public boolean canMerge();
    
  // 当多个窗口需要合并时调用，多个窗口中触发器的状态也要被合并  
  // Called when several windows have been merged into one window 
  // and the state of the triggers needs to be merged
  public void onMerge(W window, OnMergeContext ctx);
    
  // 这个方法会在一个窗口被清除时调用，它应该清除触发器自定义的各种  
  // Clears any state that the trigger might hold for the given window
  // This method is called when a window is purged
  public abstract void clear(W window, TriggerContext ctx);

  // 用于触发器中方法的上下文对象  
  // A context object that is given to Trigger methods to allow them
  // to register timer callbacks and deal with state
  public interface TriggerContext {
      // 获取当前处理时间
      // Returns the current processing time
      long getCurrentProcessingTime();
      // 获取当前水位线
      // Returns the current watermark time
      long getCurrentWatermark();
      // 注册处理时间计时器
      // Registers a processing-time timer
      void registerProcessingTimeTimer(long time);
      // 注册事件时间计时器
      // Registers an event-time timer
      void registerEventTimeTimer(long time);
      // 删除处理时间计时器
      // Deletes a processing-time timer
      void deleteProcessingTimeTimer(long time);
      // 删除事件时间计时器
      // Deletes an event-time timer
      void deleteEventTimeTimer(long time);
      // 获取一个作用域为触发器键值和当前窗口的状态对象
      // Retrieves a state object that is scoped to the window and the key of the trigger
      <S extends State> S getPartitionedState(StateDescriptor<S, ?> stateDescriptor);
  }

  // 用于onMerge方法的特殊上下文
  // Extension of TriggerContext that is given to the Trigger.onMerge() method
  public interface OnMergeContext extends TriggerContext {
      // Merges per-window state of the trigger
      // The state to be merged must support merging
      void mergePartitionedState(StateDescriptor<S, ?> stateDescriptor);
  }
    
}
````



但是有两点要特别注意：**状态清理**和**合并触发器**

1. 状态清理：
   - 在**触发器中**使用**单窗口状态**时，需要确保在删除窗口时正确删除该状态。否则，算子将**随着时间积累越来越多的状态**。
   - 为了在删除窗口时清除所有状态，**触发器的clear()方法**需要**删除所有自定义的单窗口状态**，并使用TriggerContext对象**删除所有计时器**。
2. 合并触发器
   - 在处理合并的时候，一定要注意**合并** **触发器的自定义状态**(onMerge())



下面我们举一个自定义触发器的例子

````scala
/** 一个会每隔1s提前触发一次计算的触发器 */
class OneSecondIntervalTrigger extends Trigger[SensorReading, TimeWindow] {

  // 处理每个事件  
  override def onElement(r: SensorReading, timestamp: Long,
      window: TimeWindow, ctx: Trigger.TriggerContext): TriggerResult = {

    // firstSeen是一个Boolean类型的状态，初始值为false  
    // firstSeen will be false if not set yet
    val firstSeen: ValueState[Boolean] = ctx.getPartitionedState(
      new ValueStateDescriptor[Boolean]("firstSeen", classOf[Boolean]))
      
	// 当第一个事件到达时，注册两个计时器
    // register initial timer only for first element
    if (!firstSeen.value()) {
      // compute time for next early firing by rounding watermark to second
      val t = ctx.getCurrentWatermark + (1000 - (ctx.getCurrentWatermark % 1000))
      // 注册第一个计时器，当前水位线+1s
      ctx.registerEventTimeTimer(t)
      // 注册第二个计时器，当前窗口的结束时间
      // register timer for the window end
      ctx.registerEventTimeTimer(window.getEnd)
      // 更新firstSeen状态为true  
      firstSeen.update(true)
    }
    // 返回Continue，意思是什么都不做  
    // Continue. Do not evaluate per element
    TriggerResult.CONTINUE
  }

  // 当事件时间计时器触发时，这个方法被调用  
  override def onEventTime(
      timestamp: Long,
      window: TimeWindow,
      ctx: Trigger.TriggerContext): TriggerResult = {
    // 如果是窗口的结束时间的那个计时器
    if (timestamp == window.getEnd) {
      // 执行计算并且清除窗口  
      // final evaluation and purge window state
      TriggerResult.FIRE_AND_PURGE
   // 如果+1s的计时器 
   } else {
      // register next early firing timer
      // 先注册下一个计时器，还是+1s
      val t = ctx.getCurrentWatermark + (1000 - (ctx.getCurrentWatermark % 1000))
      if (t < window.getEnd) {
        ctx.registerEventTimeTimer(t)
      }
      // 执行计算  
      // fire trigger to evaluate window
      TriggerResult.FIRE
    }
  }

  // 当处理时间计时器触发时，这个方法被调用，没啥用    
  override def onProcessingTime(
      timestamp: Long,
      window: TimeWindow,
      ctx: Trigger.TriggerContext): TriggerResult = {
    // Continue. We don't use processing time timers
    TriggerResult.CONTINUE
  }

  // 当窗口要被删除时，这个方法被调用，我们需要手动清理firstSeen状态  
  override def clear(window: TimeWindow, ctx: Trigger.TriggerContext): Unit = {

    // clear trigger state
    val firstSeen: ValueState[Boolean] = ctx.getPartitionedState(
      new ValueStateDescriptor[Boolean]("firstSeen", classOf[Boolean]))
      
    firstSeen.clear()
  }
}
````



##### 6.3.4.4 移除器



在Flink的窗口机制中，**移除器**是一个**可选组件**。它可以在**窗口函数**计算**之前**或**之后** **删除窗口中的元素**。



下面示例展示了`Evictor`接口的源码

```java
public interface Evictor<T, W extends Window> extends Serializable {
    
  // Optionally evicts elements. Called before windowing function.
  void evictBefore(Iterable<TimestampedValue<T>> elements, int size, 
   				   W window, EvictorContext evictorContext);
    
  // Optionally evicts elements. Called after windowing function.
  void evictAfter(Iterable<TimestampedValue<T>> elements, int size, 
    W window, EvictorContext evictorContext);

  // A context object that is given to Evictor methods.
  interface EvictorContext {
  	// Returns the current processing time.
  	long getCurrentProcessingTime();
  	// Returns the current event time watermark.
  	long getCurrentWatermark();
  }
}
```



### 6.4 Joining Streams on Time



在处理流时，一个常见的需求是**connect** or **join** the events of two streams。Flink的DataStream API提供了两个内置的算子：**Interval join** 和 **Window join**。在本节中，我们将描述这两个算子。



#### 6.4.1 Interval Join



Interval Join对于两个流中拥有**相同的键**，并且彼此之间的**时间戳间隔** **不超过指定的间隔**的事件进行join操作。



下图显示的两条流A和B。B中某个事件会与A中的一些事件join成对，**具体步骤**如下

- 以B中某个事件为**基事件**
- **从A中选择**那些相较于基事件，**时间戳间隔**在$[-1h, +15min]$范围内的事件
- 将这些事件**join成事件对**，来**一起处理**。如下图的**a事件**和**b事件**就会组成**事件对**，**a事件**和**c事件**也会组成**事件对**

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201111125659.png)



Interval Join的API使用方法如下

```scala
input1
  .keyBy(…) // 按键分区
  .between(Time.hour(-1), Time.minute(15)) // 指定事件的上下界
  .process(ProcessJoinFunction) // JOIN成功的事件对会被发送给这个函数，由它来处理
```



#### 6.4.2 Window Join



顾名思义，Window Join是**基于**Flink的**窗口机制**的。两个输入流的元素都被分配到**同一个公共窗口**，并在这个窗口收集完成时进行叉乘积联接，然后交给ProcessJoinFunction计算。



工作流程如下图所示

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201111130527.png)



上图的解释如下

- 两个输入流都**根据**它们各自的**键**属性进行**分区**，
- **公共窗口分配器**将两个流的事件映射到**公共窗口**，这意味着**公共窗口同时存储着来自两条输入流的事件**。
- 当**窗口的触发器触发**时，将对两个输入流中的每个元素**组合**（叉乘积）**调用JoinFunction**。
- 此外还可以**自定义** **触发器**和**移除器**。由于这两个流的事件被映射到相同的窗口中，因此触发器和移除器的行为与常规窗口算子中的触发器和移除器行为完全相同。



下面来看看我们怎么使用这个API

````scala
input1.join(input2)
  .where(...)       // specify key attributes for input1 指定第一条流的key状态
  .equalTo(...)     // specify key attributes for input2 指定第二条流的key状态
  .window(...)      // specify the WindowAssigner 指定窗口分配器
 [.trigger(...)]    // optional: specify a Trigger 指定触发器(可以不指定)
 [.evictor(...)]    // optional: specify an Evictor 指定移除器(可以不指定)
  .apply(...)       // specify the JoinFunction 指定处理函数
````



除了Join之外，还可以使用cogroup()。Join和**CoGroup**的总体逻辑是相似的，Join对来自两个输入的每一对事件调用JoinFunction，而CoGroup对窗口中两条输入序列的遍历器调用GroupFunction。（也就是说参数不同）



### 6.5 处理迟到数据



**迟到事件**是在**算子需要执行的计算已经完成时** **到达算子**的事件。在**事件时间窗口算子**这种情况下，如果事件到达算子，但是窗口分配器将其分配到了一个**已经完成了计算的窗口**（也就是算子的水位线超过了窗口的结束时间的窗口），则该事件就是迟到的。



DataStream API提供了三种处理迟到事件的方案:

- Dropping: 简单地**丢弃**迟到事件。
- Redirect: 将迟到事件**重定向到单独的流**中。
- Update: 根据迟到事件**更新计算结果**，并且发出结果。



下面三小节分别介绍这三种情况



#### 6.5.1 丢弃迟到事件(Dropping)



处理迟到事件最简单的方法就是丢弃。这也是事件时间窗口的**默认行为**。因此，迟到的元素将不会创建新窗口。



#### 6.5.2 重定向迟到事件(Redirect)



迟到事件还可以使用副输出特性重定向到另一个DataStream，这样就可以根据业务需求来进行各种不同种类的后期处理



下面举个例子，说明如何**将迟到事件重定向到副输出**

```scala
// 处理正常流
val filteredReadings: DataStream[SensorReading] = readings
	.process(new LateReadingsFilter)

// 取出副输出
val lateReadings: DataStream[SensorReading] = filteredReadings
	.getSideOutput(lateReadingsOutput)

// 对正常流进行后续处理 print the filtered stream
filteredReadings.print()

// 对副输出进行后续处理 print messages for late readings
lateReadings
	.map(r => "*** late reading *** " + r.id)
	.print()

/** A ProcessFunction that filters out late sensor readings and re-directs them to a side output */
class LateReadingsFilter extends ProcessFunction[SensorReading, SensorReading] {

  override def processElement(
      r: SensorReading,
      ctx: ProcessFunction[SensorReading, SensorReading]#Context,
      out: Collector[SensorReading]): Unit = {

    // compare record timestamp with current watermark
    if (r.timestamp < ctx.timerService().currentWatermark()) {
      // this is a late reading => redirect it to the side output
      // 迟到的事件重定向到副输出中  
      ctx.output(LateDataHandling.lateReadingsOutput, r)
    } else {
      // 正常的事件直接输出
      out.collect(r)
    }
  }
}
```



#### 6.5.3 基于迟到事件更新结果(Update)



另一种策略是重新计算结果并发出更新。但是，为了能够重新计算和更新结果，需要考虑一些问题。

- 支持Update策略的算子需要在第一次**结果发出后** **保存**计算所需的所有**状态**。
- **下游算子**或**外部系统** **能够处理**得了这些更新。



窗口算子API提供了一个方法来显式声明你希望能够处理迟到事件。在使用事件时间窗口时，可以指定一个额外的时间段，称为**延迟容忍度**(allowed lateness)。配置了该属性的窗口不会被立刻删除，而是会被保存到延迟容忍度再删除。



下面举个例子，来看看**延迟容忍度**怎么使用

````scala
val readings: DataStream[SensorReading] = ???
val countPer10Secs: DataStream[(String, Long, Int, String)] = readings
	.keyBy(_.id)
  	.timeWindow(Time.seconds(10))
  	// process late readings for 5 additional seconds 设置延迟容忍度为5s
  	.allowedLateness(Time.seconds(5))
  	// count readings and update results if late readings arrive
  	.process(new UpdatingWindowCountFunction)

/**这个处理函数会采用Update策略处理迟到事件*/
class UpdatingWindowCountFunction extends ProcessWindowFunction[SensorReading,
	(String, Long, Int, String), String, TimeWindow] {
  
  override def process(
      id: String,
      ctx: Context,
      elements: Iterable[SensorReading],
      out: Collector[(String, Long, Int, String)]): Unit = {
    // count the number of readings
    val cnt = elements.count(_ => true)
    // 这个状态用来标记是否是首次计算
    val isUpdate = ctx.windowState.getState(
      new ValueStateDescriptor[Boolean]("isUpdate", Types.of[Boolean]))
    if (!isUpdate.value()) {
      // 首次计算并发出结果
      out.collect((id, ctx.window.getEnd, cnt, "first"))
      isUpdate.update(true)
    } else {
      // 不是首次计算，发出更新
      out.collect((id, ctx.window.getEnd, cnt, "update"))
    }
  }
}
````

