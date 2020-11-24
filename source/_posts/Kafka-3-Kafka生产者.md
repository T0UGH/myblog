---
title: '[Kafka][3][Kafka生产者]'
date: 2020-11-24 20:52:20
tags:
    - 大数据
    - Kafka
    - 消息队列
categories:
    - Kafka
---
## 第3章 Kafka生产者——向Kafka写入数据



> 对应代码仓库地址:<https://github.com/T0UGH/getting-started-kafka>
>
> 参考资料:
>
> **kafka权威指南**<https://book.douban.com/subject/27665114/>
>
> 大数据通用的序列化器——Apache Avro<https://www.jianshu.com/p/0a85bfbb9f5f>
>
> Kafka 中使用 Avro 序列化框架(一)<https://www.jianshu.com/p/4f724c7c497d>
>
> Kafka 中使用 Avro 序列化组件(三)：Confluent Schema Registry<https://www.jianshu.com/p/cd6f413d35b0>



本章对应代码仓库可查看<https://github.com/T0UGH/getting-started-kafka/tree/main/src/main/java/cn/edu/neu/demo/ch3>



在这一章，我们将从**Kafka生产者**的**设计**和**组件**讲起，学习如何使用Kafka 生产者。

1. 我们将演示如何创建KafkaProducer和ProducerRecords对象、
2. 如何将记录**发送**给Kafka
3. 如何处理从Kafka返回的错误
4. 介绍用于控制生产者行为的重要**配置**选项
5. 深入探讨如何使用不同的**分区**方法和**序列化器**，以及如何自定义序列化器和分区器。



### 3.1 生产者概览



先展示向Kafka发送消息的主要步骤

![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201123193012.png)

1. 首先**创建**一个**ProducerRecord**对象开始，ProducerRecord 对象需要包含目标**主题**和要发送的**内容**，有可能还包含**键**和**分区**信息

2. 把ProducerRecord中的键和值**序列化**成字节数组，这样它们才能够在网络上传输

3. 接下来，数据被传给**分区器**。

   - 如果之前在ProducerRecord对象里**指定了分区**，那么分区器就不会再做任何事情，直接把指定的分区**返回**。
   - 如果**没有指定分区**，那么分区器会**根据**ProducerRecord对象的**键**来选择一个**分区**。

4. 紧接着，这条记录被添加到一个**记录批次**里，这个批次里的所有消息会被发送到相同的主题和分区上。

5. 有一个**独立的线程**负责把这些记录批次**发送**到相应的broker 上。

6. **服务器**在收到这些消息时会返回一个**响应**。

   - 如果消息**成功**写入Kafka，就返回一个**RecordMetaData**对象，它包含了**主题**和**分区**信息，以及记录在分区里的**偏移量**。

   - 如果写入**失败**，则会返回一个**错误**。生产者在收到错误之后会尝试**重新发送**消息，几次之后如果还是失败，就返回错误信息。



### 3.2 创建Kafka生产者



下面展示如何创建一个KafkaProducer

````java
Properties kafkaProperties = new Properties();

// fixme: 运行时请修改47.94.139.116:9092为自己的kafka broker地址
kafkaProperties.put("bootstrap.servers", "47.94.139.116:9092");

kafkaProperties.put("key.serializer",                     "org.apache.kafka.common.serialization.StringSerializer");

kafkaProperties.put("value.serializer",                           "org.apache.kafka.common.serialization.StringSerializer");

kafkaProperties.put("acks", "all");

// 根据配置创建Kafka生产者
KafkaProducer<String, String> kafkaProducer 
    = new KafkaProducer<String, String>(kafkaProperties);

````



在创建Kafka生产者之前，有3个必选的配置项



| 配置项            | 解释                                                         |
| ----------------- | ------------------------------------------------------------ |
| bootstrap.servers | 该属性指定broker 的地址清单，地址的格式为host:port, host:port。 |
| key.serializer    | key.serializer 必须被设置为一个实现了org.apache.kafka.common.serialization.Serializer 接口的类，生产者会使用这个类**把键序列化成字节数组**。Kafka 客户端默认提供了ByteArraySerializer、StringSerializer 和IntegerSerializer |
| value.serializer  | 与key.serializer 一样，value.serializer 指定的类会**将值序列化**。 |



### 3.3 发送消息到Kafka



实例化生产者对象后，接下来就可以开始发送消息了。发送消息主要有以下3 种方式：



- **发送并忘记**（fire-and-forget）
  - 我们把消息发送给服务器，但并**不关心它是否正常到达**。
  - 大多数情况下，消息会正常到达，因为Kafka 是高可用的，而且生产者会自动尝试重发。
  - 不过，使用这种方式有时候也会**丢失一些消息**。
- **同步发送**
  - 我们使用send() 方法发送消息，它会返回一个**Future 对象**，**调用get() 方法进行等待**，就可以知道消息是否发送成功。
- **异步发送**
  - 我们调用send() 方法，并指定一个**回调函数**，服务器在返回响应时调用该函数。



下面分别演示这三种方式



#### 3.3.1 发送并忘记



````java
ProducerRecord<String, String> producerRecord = new ProducerRecord<String, String>(
	"sun", "s1", "cn dota best dota");

// 发送消息
try{
    kafkaProducer.send(producerRecord);
}catch(Exception e){
    e.printStackTrace();
}
````

- 生产者的send() 方法将ProducerRecord对象作为参数。ProducerRecord中包含**目标主题**的名字和要发送的**键**对象和**值**对象。
- 我们使用生产者的**send()方法**发送ProducerRecord对象。send() 方法会**返回**一个包含RecordMetadata 的**Future对象**，不过因为我们这里忽略了返回值，所以无法知道消息是否发送成功。如果不关心发送结果，那么可以使用这种发送方式。比如，记录日志。
- 虽然我们忽略返回值的同时也忽略了返回的异常。不过在**发送消息之前**，生产者还是有可能发生其他的**异常**，这些异常将被抛出。这些异常有可能是：
  - SerializationException（说明序列化消息失败）
  - BufferExhaustedException 或TimeoutException（说明缓冲区已满）
  - 又或者是InterruptException（说明发送线程被中断）。



#### 3.3.2 同步发送



````java
ProducerRecord<String, String> producerRecord = new ProducerRecord<String, String>(
	"sun", "s1", "cn dota best dota");

// 发送消息
try{
    kafkaProducer.send(producerRecord).get();
}catch(Exception e){
    e.printStackTrace();
}
````

- producer.send()方法先返回一个Future对象，然后调用Future 对象的**get()方法**会**阻塞当前线程**来等待Kafka响应。
  - 如果服务器返回一个**错误**，get() 方法会**抛出异常**。
  - **否则**，我们会得到一个**RecordMetadata**对象，可以通过它获取消息的**主题**、**分区**、**偏移量**等信息。


-  如果在**发送消息之前**或者在**发送消息的过程中**发生了任何**错误**，比如broker 返回了一个不允许重发消息的异常或者已经超过了重发的次数，那么就会**抛出异常**。



KafkaProducer一般会发生两类错误。

- 一类是**可重试错误**，这类错误可以通过重发消息来解决。比如对于连接错误，可以通过再次建立连接来解决，“无主（no leader）”错误则可以通过重新为分区选举首领来解决。KafkaProducer 可以被配置成自动重试，如果在**多次重试后仍无法解决问题**，应用程序会收到一个**重试异常**。
- 另一类错误**无法通过重试解决**，比如“消息太大”异常。对于这类错误，KafkaProducer**直接抛出异常**。



#### 3.3.3 异步发送



如果只发送消息而不等待响应，那么可以**避免阻塞**线程来等待，从而提高发送效率。



大多数时候，我们并不需要等待响应——尽管Kafka会把目标主题、分区信息和消息的偏移量发送回来，但对于发送端的应用程序来说不是必需的。不过在**遇到消息发送失败时**，我们需要**抛出异常**、**记录错误日志**，或者把消息**写入**“错误消息”**文件**以便日后分析。



这时我们可以为send()方法**注册一个回调函数**，让它来处理异步调用的返回结果



````java
// 创建ProducerRecord，它是一种消息的数据结构
ProducerRecord<String, String> producerRecord 
    = new ProducerRecord<String, String>("sun", "s1", "cn dota best dota");

// 发送消息
kafkaProducer.send(producerRecord, new Callback() {
    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
        if(e != null){
            e.printStackTrace();
        }else{
            System.out.println(recordMetadata);
        }   
    }
});
````

- 这里通过注册一个**回调函数**来**处理异步**发送的结果
  - 如果错误，则`onCompletion`方法的`Exception`参数不为null，我们可以针对这个异常进行处理
  - 如果没有错误，`RecordMetadata`不为null，我们可以从中获取主题信息、分区信息、偏移量信息



### 3.4 生产者的配置



生产者还有很多可配置的参数，在Kafka 文档里都有说明，它们大部分都有合理的默认值，所以没有必要去修改它们。不过有几个参数在内存使用、性能和可靠性方面对生产者影响比较大，接下来我们会一一说明。



#### 3.4.1 acks



**acks**参数指定了必须**要有多少个分区副本收到消息**，**生产者**才会**认为消息写入是成功**的。



- 如果acks=0，**生产者发送消息之后就立刻认为消息写入成功**。
  - 也就是说，如果服务器没有收到消息，生产者也无从得知，消息也就丢失了。
  - 不过，因为**生产者不需要等待服务器的响应**，所以它可以以网络能够支持的最大速度发送消息，从而达到很高的**吞吐量**。

- 如果acks=1，只要**集群的首领节点收到消息**，生产者就会**收到**一个来自服务器的**成功响应**。
  - 如果消息无法到达首领节点（比如首领节点崩溃，新的首领还没有被选举出来），生产者会收到一个错误响应，为了避免数据丢失，生产者会重发消息。
- 如果acks=all，只有当**首领节点**和**所有复制节点**全部**收到消息**时，生产者才会收到一个来自服务器的**成功响应**。
  - 这种模式是**最安全**的。
  - 不过，它的**延迟**比acks=1 时更高，因为我们要等待不只一个服务器节点接收消息。



#### 3.4.2 retries(重试次数)



retries 参数的值决定了**生产者可以重发消息的次数**，如果达到这个次数，生产者会放弃重试并返回错误。默认情况下，生产者会在每次**重试之间等待**100ms。



因为**生产者会自动进行重试**，所以就**没必要**在代码逻辑里**处理那些可重试的错误**。你只需要处理那些不可重试的错误或重试次数超出上限的情况。



#### 3.4.3 batch.size(批次大小)和linger.ms(批次等待时间)



KafkaProducer会在**批次填满**或linger.ms**达到上限**时把批次**发送**出去。



该参数指定了一个**批次可以使用的内存大小**，**按照字节数计算**（而**不是**消息**个数**）。当批次被填满，批次里的所有消息会被发送出去。



该参数指定了生产者在发送批次之前**等待更多消息加入批次的时间**



#### 3.4.4 max.in.flight.requests.per.connection

该参数指定了**生产者**在**收到服务器响应之前可以发送多少个消息**。

- 它的值越高，就会占用越多的内存，不过也会提升吞吐量。
- 把它**设为1**可以保证消息是按照发送的**顺序写入**服务器的，即使发生了重试。



### 3.5 序列化器



我们已经在之前的例子里看到，创建一个生产者对象必须指定序列化器。我们已经知道如何使用默认的字符串序列化器，Kafka 还提供了整型和字节数组序列化器，不过它们还不足以满足大部分场景的需求。到最后，我们需要序列化的记录类型会越来越多。



接下来演示如何开发**自定义序列化器**，并介绍**Avro序列化器**。如果**发送到Kafka的对象** **不是**简单的**字符串**或**整型**，那么可以使用**序列化框架**来创建消息记录，如Avro、Thrift 或Protobuf，或者使用自定义序列化器。我们强烈**建议使用通用的序列化框架**。



#### 3.5.1 自定义序列化器



```java
/**
 * 一个简单的pojo，为了演示如何自定义序列化器
 * */
public class Customer {

    private int customerId;

    private String customerName;

    public Customer(int customerId, String customerName) {
        this.customerId = customerId;
        this.customerName = customerName;
    }

    public Customer() {
    }

    public int getCustomerId() {
        return customerId;
    }

    public void setCustomerId(int customerId) {
        this.customerId = customerId;
    }

    public String getCustomerName() {
        return customerName;
    }

    public void setCustomerName(String customerName) {
        this.customerName = customerName;
    }
}

```



````java
/**
 * 自定义序列化器
 * */
public class CustomerSerializer implements Serializer<Customer> {
    public void configure(Map<String, ?> configs, boolean isKey) {
        // 不需要配置任何
    }

    /**
     * Customer对象的序列化函数，组成如下
     * 前4字节: customerId
     * 中间4字节: customerName字节数组长度
     * 后面n字节: customerName字节数组
     * */
    public byte[] serialize(String topic, Customer data) {
        try{
            byte[] serializedName;
            int stringSize;
            if(data == null){
                return null;
            }else{
                if(data.getCustomerName() != null){
                    serializedName = data.getCustomerName().getBytes("utf-8");
                    stringSize = serializedName.length;
                } else{
                    serializedName = new byte[0];
                    stringSize = 0;
                }
            }
            ByteBuffer buffer = ByteBuffer.allocate(4 + 4 + stringSize);
            buffer.putInt(data.getCustomerId());
            buffer.putInt(stringSize);
            buffer.put(serializedName);
            return buffer.array();
        } catch(Exception e){
            throw new SerializationException(
                "Error when serializing Customer to byte[] " + e);
        }
    }

    public byte[] serialize(String topic, Headers headers, Customer data) {
        return serialize(topic, data);
    }

    public void close() {
        // 不需要关闭任何
    }
}

````



不过我们不建议采用自定义序列化器，理由如下

- 如果我们有多种类型的消费者，可能需要把customerID 字段变成长整型，或者为Customer 添加startDate 字段，这样就会出现**新旧消息的兼容性问题**。
- 在不同版本的序列化器和反序列化器之间**调试兼容性**问题着实是个**挑战**——你需要比较原始的字节数组。
- 更糟糕的是，如果同一个公司的**不同团队**都需要往Kafka 写入Customer 数据，那么他们就需要使用相同的序列化器，如果**序列化器发生改动**，**他们几乎都要在同一时间修改代码**。



#### 3.5.2 使用Avro序列化



Apache Avro是一种与**编程语言无关**的**序列化格式**。



**Avro数据**通过与语言无关的**schema来定义**。

- **schema通过JSON来描述**，
- 数据可以被序列化成二进制文件或JSON 文件，不过一般会使用二进制文件。
- Avro 在**读写文件**时**需要用到schema**，schema 一般会被**内嵌**在数据文件里。



Avro 有一个很有意思的特性是，当负责**写消息的应用程序**使用了**新版本的schema**，负责**读消息的应用程序**可以继续处理消息而**无需做任何改动**，这个特性使得它特别适合用在像Kafka 这样的消息系统上。



下面举个例子

> 假设我们有一个v0.0.1的schema
>
> ````json
> {
> "namespace": "customerManagement.avro",
> 	"type": "record",
> 	"name": "Customer",
> 	"fields": [
> 		{"name": "id", "type": "int"},
> 		{"name": "name", "type": "string"},
> 		{"name": "faxNumber", "type": ["null", "string"], "default": "null"} 
> 	]
> }
> ````
>
> 
>
> 过了一段时间，我们需要删除`faxnumber`字段(传真号码)，添加一个`email`字段，新的schema(v0.0.2)如下
>
> ````json
> {
> "namespace": "customerManagement.avro",
> 	"type": "record",
> 	"name": "Customer",
> 	"fields": [
> 		{"name": "id", "type": "int"},
> 		{"name": "name", "type": "string"},
> 		{"name": "email", "type": ["null", "string"], "default": "null"} 
> 	]
> }
> ````
>
> 
>
> 更新到新版的schema 后:
>
> - 旧记录仍然包含faxNumber 字段
> - 而新记录则包含email 字段
> - **部分**负责**读取数据的应用程序**进行了**升级**，那么它们是如何处理这些变化的呢？
>
> 
>
> 在**消费者升级之前**：
>
> - 它们会调用类似getName()、getId() 和getFaxNumber() 这样的方法。
> - 如果碰到使用新schema 构建的消息，getName() 和getId() 方法仍然能够正常返回，但getFaxNumber() 方法会返回null，因为消息里不包含传真号码。
>
> 
>
> 在**消费者升级之后**
>
> - getEmail() 方法取代了getFaxNumber() 方法。
> - 如果碰到一个使用旧schema 构建的消息，那么getEmail() 方法会返回null，因为旧消息不包含邮件地址。
>
> 
>
> 现在可以看出使用Avro 的好处了：我们**修改了消息的schema**，但并**不需要更新所有消费者**，而这样仍然不会出现异常或阻断性错误，也不需要对现有数据进行大幅更新。



#### 3.5.3 在Kafka中使用Avro



如果在每条Kafka 记录里都嵌入schema，会让记录的大小成倍地增加。

- 但是在**读取记录**时仍然**需要用到整个schema**，所以要先找到schema。
- 我们要使用“**schema 注册表**”来达到目的。
- Kafka中并不包含schema注册表的实现，现在已经有一些开源的schema注册表实现，比如：Confluent Schema Registry。
- 我们把所有写入数据需要用到的**schema保存在注册表里**，然后在**记录里放一个schema的标识符**。
- **消费者**使用记录中的标识符**从注册表里拉取schema**来反序列化记录。



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201124195244.png)



### 3.6 分区



在之前的例子里，**ProducerRecord对象**包含了目标**主题**、**键**和**值**。Kafka 的**消息**是一个个**键值对**，ProducerRecord 对象**可以只包含目标主题和值**，**键**可以设置为默认的**null**，不过大多数应用程序会用到键。



键有两个用途：

1. 可以作为消息的**附加信息**，
2. 也可以用来**决定消息该被写到主题的哪个分区**。拥有相同键的消息将被写到同一个分区。



如果要创建键为null的消息，不指定键就可以

````java
ProducerRecord<Integer, String> record = new ProducerRecord<>("CustomerCountry", "USA");
````



如果**键为null**，并且使用了默认的分区器，那么记录将被随机地发送到主题内各个可用的分区上。分区器使用**轮询（Round Robin）算法**将消息均衡地分布到各个分区上。



如果**键不为空**，并且使用了默认的分区器，那么Kafka会使用**Kafka自己的散列算法**对键进行**散列**（使用Kafka 自己的散列算法，即使升级Java 版本，散列值也不会发生变化），然后根据散列值把消息**映射到特定的分区**上。这里的关键之处在于，**同一个键总是被映射到同一个分区上**。



只有在**不改变主题分区数量的情况下**，键与分区之间的映射才能保持不变。举个例子，在分区数量保持不变的情况下，可以保证用户045189 的记录总是被写到分区34。在从分区读取数据时，可以进行各种优化。不过，一旦主题增加了新的分区，这些就无法保证了——旧数据仍然留在分区34，但新的记录可能被写到其他分区上。**如果要使用键来映射分区，那么最好在创建主题的时候就把分区规划好，而且永远不要增加新分区。**



#### 3.6.1 自定义分区器



**默认情况下**，kafka自动创建的主题的**分区数量为1**，所以我们需要先修改分区数量，来让自定义分区器有点用。

1. 先cd到`opt\kafka\bin\`

2. 运行命令：

   ````shell
   ./kafka-topics.sh --zookeeper 47.94.139.116:2181/kafka --alter --topic sun --partitions 4
   ````

3. 查看是否修改成功

   ```shell
   ./kafka-topics.sh --describe --zookeeper 47.94.139.116:2181/kafka --topic sun
   ```



![](https://healthlung.oss-cn-beijing.aliyuncs.com/20201124201605.png)



或者也可以通过修改`server.properties`中的`nums.partions`并重启，来更改默认分区数量。



下面自定义一个分区器，它根据键来划分分区：若键为Banana则放入最后一个分区，若键不为Banana则散列到其他分区。



````java
/**
 * 自定义分区器
 * 若键为Banana则放入最后一个分区，若键不为Banana则散列到其他分区
 * */
public class BananaPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {

        // 分区信息列表
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);

        //分区数量
        int partitionsAmount = partitions.size();

        // 如果只有一个分区，那全都放到partition0就完事了
        if(partitionsAmount == 1){
            return 0;
        }

        if(keyBytes == null || ! (key instanceof String)){
            throw new InvalidRecordException("We expect all messages to have customer name as key");
        }

        if(key.equals("Banana")){
            return partitionsAmount - 1;
        }

        return (Math.abs(Utils.murmur2(keyBytes)) % (partitionsAmount - 1));

    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> configs) {

    }
}
````



然后，在producer配置中加入`kafkaProperties.put("partitioner.class", "cn.edu.neu.demo.ch3.partitioner.BananaPartitioner");`即可。









