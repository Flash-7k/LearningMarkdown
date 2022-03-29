# 1. 概述

**定义**

+ 传统定义：Kafka是一个**分布式**的基于**发布/订阅模式**的**消息队列**（Message Queue），主要应用于大数据实时处理领域。
  + 发布/订阅模式：消息分为多种类型，订阅者根据需求，选择性订阅
+ 最新定义： Kafka 是一个开源的**分布式事件流平台**（ Event Streaming Platform），被数千家公司用于**高性能数据管道**、**流分析**、**数据集成**和**关键任务应用**。

**应用场景**

+ **缓存/消峰**：控制和优化数据流经过系统的速度，解决生产消息和消费消息的处理速度不一致的情况。
+ **解耦：**允许你独立的扩展或修改两边的处理过程，只要确保它们遵守同样的接口约束。
+ **异步通信：**允许用户把一个消息放入队列，但并不立即处理它，然后在需要的时候再去处理它们。

**消息队列的两种模式**

+ 点对点
  + 消费者主动拉取数据，消息收到后清除消息
+ 发布订阅
  + 可以有多个topic主题（浏览、点赞、收藏、评论等）
  +  消费者消费数据之后，不删除数据
  +  每个消费者相互独立，都可以消费到数据

**Kafka基础架构**

+ 生产者：消息生产者，向 Kafka Broker 发消息的客户端
+ Broker：一台 Kafka 服务器就是一个 Broker。一个集群由多个 Broker组成。一个Broker可以容纳多个 Topic
  + Topic：可以理解为一个队列，生产者和消费者面向的都是一个 Topic。为了实现扩展性，一个非常大的 Topic可以分布到多个 Broker（即服务器）上，一个 Topic可以分为多个 Partition。
  + Partition：每个Partition是一个有序的队列。
  + Replica：副本。一个 Topic 的每个 Partition 都有若干个副本，一个 Leader 和若干 Follower
  + Leader：每个分区多个副本的”主”，生产者发送数据的对象，以及消费者消费数据的对象
  + Follower：每个分区多个副本中的“从”，实时从 Leader 中同步数据，保持和 Leader 数据的同步。Leader 发生故障时，某个 Follower 会成为新的 Leader
+ 消费者：消息消费者，向Kafka Broker 拉取消息的客户端
  + 消费者组，由多个消费者组成。**消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费；消费者组之间互不影响。**所有的消费者都属于某个消费者组，即**消费者组是逻辑上的一个订阅者**

# 2. 安装及命令行操作

## 1）安装

**集群规划**

| hadoop102 | hadoop103 | hadoop104 |
| --------- | --------- | --------- |
| zk        | zk        | zk        |
| kafka     | kafka     | kafka     |

**集群部署**

+ 解压安装包

```shell
tar	-zxvf	kafka_2.12-3.0.0.tgz -C /opt/module/
```

+ 进入`/opt/module/kafka/config` 目录，修改配置文件`server.properties`

```properties
#修改 broker.id、log.dirs、zookeeper.connect



#broker 的全局唯一编号，不能重复，只能是数字。
broker.id=0                            #必须修改！！

#处理网络请求的线程数量
num.network.threads=3
#用来处理磁盘 IO 的线程数量
num.io.threads=8
#发送套接字的缓冲区大小
socket.send.buffer.bytes=102400
#接收套接字的缓冲区大小
socket.receive.buffer.bytes=102400
#请求套接字的缓冲区大小
socket.request.max.bytes=104857600

#kafka 运行日志(数据)存放的路径，路径不需要提前创建，kafka 自动帮你创建，可以配置多个磁盘路径，路径与路径之间可以用"，"分隔
log.dirs=/opt/module/kafka/datas                 #自定义数据路径！！！

#topic 在当前 broker 上的分区个数
num.partitions=1
#用来恢复和清理 data 下数据的线程数量
num.recovery.threads.per.data.dir=1 
# 每个 topic 创建时的副本数，默认时 1 个副本
offsets.topic.replication.factor=1 #segment 文件保留的最长时间，超时将被删除log.retention.hours=168
#每个 segment 文件的大小，默认最大 1G 
log.segment.bytes=1073741824
# 检查过期数据的时间，默认 5 分钟检查一次是否数据过期
log.retention.check.interval.ms=300000

#配置连接Zookeeper 集群地址（在 zk 根目录下创建/kafka，方便管理） 
zookeeper.connect=hadoop102:2181,hadoop103:2181,hadoop104:2181/kafka
```

+ 分发安装包

```shell
xsync kafka/
```

+ 分别在 hadoop103 和 hadoop104 上修改配置文件`/opt/module/kafka/config/server.properties` 中的 `broker.id=1`、`broker.id=2`
+ 配置环境变量

```shell
sudo vim /etc/profile.d/my_env.sh
```

```sh
#KAFKA_HOME
export KAFKA_HOME=/opt/module/kafka
export PATH=$PATH:$KAFKA_HOME/bin
```

+ 使文件生效。并分发到其他节点，source

```shell
source /etc/profile
```

**集群启停**

+ 先启动zookeeper集群，然后启动Kafka

```shell
zk.sh start #自己写的zk启动脚本

#在102 103 104上分别启动kafka
/opt/module/kafka/bin/kafka-server-start.sh -daemon /opt/module/kafka/config/server.properties
```

+ 关闭集群

```shell
/opt/module/kafka/bin/kafka-server-stop.sh
```

+ 集群启停脚本：`/home/atguigu/bin/kf.sh`

```sh
#! /bin/bash

case $1 in 
"start"){
    for i in hadoop102 hadoop103 hadoop104 do
    echo " --------启动 $i Kafka	"
    ssh	$i	"/opt/module/kafka/bin/kafka-server-start.sh	-daemon /opt/module/kafka/config/server.properties"
    done
};;
"stop"){
    for i in hadoop102 hadoop103 hadoop104 do
    echo " --------停止 $i Kafka	"
    ssh $i "/opt/module/kafka/bin/kafka-server-stop.sh "
    done
};;
esac
```

+ 添加执行权限：`chmod +x kf.sh`

> **注意：**停止 Kafka 集群时，一定要**等 Kafka 所有节点进程全部停止**后再停止 Zookeeper 集群。因为 Zookeeper 集群当中记录着 Kafka 集群相关信息，Zookeeper 集群一旦先停止， Kafka 集群就没有办法再获取停止进程的信息，只能手动杀死Kafka 进程了

## 2）Kafka命令行操作

### 主题命令行操作

+ 查看操作主题命令参数

```shell
/opt/module/kafka/bin/kafka-topic.sh
```

| 参数                                               | 描述                                  |
| -------------------------------------------------- | ------------------------------------- |
| --bootstrap-server  <String: server : port>        | 连接的 Kafka  Broker 主机名称和端口号 |
| --topic <String: topic>                            | 操作的 topic  名称                    |
| --create                                           | 创建主题                              |
| --delete                                           | 删除主题                              |
| --alter                                            | 修改主题                              |
| --list                                             | 查看所有主题                          |
| -describe                                          | 查看主题详细描述                      |
| --partitions <Integer: # of  partitions>           | 设置分区数（修改只能增，不能减）      |
| --replication-factor<Integer:  replication factor> | 设置分区副本                          |
| --config <String: name=value>                      | 更新系统默认的配置                    |

### 生产者命令行操作

+ 查看操作生产者命令参数

```shell
/opt/module/kafka/bin/kafka-consolo-producer.sh
```

+ 样例：发送信息

```shell
bin/kafka-console-producer.sh --bootstrap-server hadoop102:9092 --topic first
>hello world
>hello flash7k
```

### 消费者命令行操作

+ 查看操作消费者命令参数

```shell
/opt/module/kafka/bin/kafka-consolo-consumer.sh
```

| 参数                                        | 描述                                  |
| ------------------------------------------- | ------------------------------------- |
| --bootstrap-server  <String: server : port> | 连接的 Kafka  Broker 主机名称和端口号 |
| --topic  <String: topic>                    | 操作的 topic  名称                    |
| --from-beginning                            | 从头开始消费                          |
| --group  <String: consumer group id>        | 指定消费者组名称                      |

# 3. 生产者

## 1）生产者消息发送流程

详情见尚硅谷笔记，发送流程图很重要！！

**发送原理**

外部数据 --> Kafka 生产者 --> 创建`main`线程和`Sender`线程

+ `main`线程将消息发送给`RecordAccumulator`

  + `send(ProducerRecord)` -->` Interceptors`拦截器 --> `Serializer`序列化器 --> `Partitioner`分区器 -->`RecordAccumulator`

  + `RecordAccumulator`：默认32m
  + `ProducerBatch`：`main`线程传来的数据，被放入`DQueue`，等待读取
  + 发送条件
    + `batch.size`：只有数据积累到`batch.size`之后，`sender`才会发送数据。默认16k
    + `linger.ms`：如果数据迟迟未达到`batch.size`，sender等待linger.ms设置的时间到了之后就会发送数据。单位ms，默认值是0ms，表示没有延迟
  + 数据发送之后就从`DQueue`中被清除

+ Sender线程不断从`RecordAccumulator`中拉取消息发送到Kafka Broker

  + Sender读取数据 --> NetworkClient --> InFlightRequests --> Selector --> 发送到Kafka Broker --> 等待应答acks，若失败则不断重试，Broker节点最多缓存5个请求
  + 应答acks
    + 0：生产者发送过来的数据，不需要等数据落盘应答
    + 1：生产者发送过来的数据，Leader收到数据后应答（通常用于普通日志文件）
    + -1（all）：生产者发送过来的数据，Leader和ISR队列里面的所有节点收齐数据后应答。（-1和all等价）

## 2）生产者重要参数

| 参数名称                                 | 描述                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| `bootstrap.servers  `                    | 生产者连接集群所需的broker地址清单<br />例如   hadoop102:9092,hadoop103:9092，可以设置1个或者多个，逗号隔开。并非需要所有的 broker 地址，因为生产者从给定的 broker里查找到其他 broker 信息 |
| `key.serializer` 和 `value.serializer  ` | 指定发送消息的 key 和  value  的序列化类型。一定要写全类名   |
| `  buffer.memory  `                      | RecordAccumulator 缓冲区总大小，默认 32m                     |
| ` batch.size  `                          | 缓冲区一批数据最大值，默认 16k。适当增加该值，可  以提高吞吐量，但是如果该值设置太大，会导致数据传输延迟增加 |
| `linger.ms`                              | 如果数据迟迟未达到 `batch.size`，sender 等待 `linger.time`  之后就会发送数据。单位 ms，默认值是 0ms，表示没有延迟。生产环境建议该值大小为 5-100ms 之间 |
| `acks`                                   | 0：生产者发送过来的数据，不需要等数据落盘应答<br />1：生产者发送过来的数据，Leader 收到数据后应答<br />-1（all）：生产者发送过来的数据，Leader+和 ISR 队列里面的所有节点收齐数据后应答。默认值是-1，-1 和all 是等价的 |
| `max.in.flight.requests.per.connection`  | 允许最多没有返回 ack 的次数，默认为 5，开启幂等性  要保证该值是 1-5 的数字 |
| `retries`                                | 当消息发送出现错误的时候，系统会重发消息。retries 表示重试次数。默认是 int 最大值，2147483647。  **如果设置了重试，还想保证消息的有序性，需要设置  `MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION=1`  否则在重试此失败消息的时候，其他的消息可能发送成功了** |
| `retry.backoff.ms`                       | 两次重试之间的时间间隔，默认是 100ms                         |
| `enable.idempotence`                     | 是否开启幂等性，默认 true，开启幂等性                        |
| `compression.type`                       | 生产者发送的所有数据的压缩方式。默认是 none，也就是不压缩。  支持压缩类型：none、gzip、snappy、lz4 和 zstd |

## 3）生产者API

总体流程：

+ 创建maven工程，在`pom.xml`中导入依赖

```xml
<dependency>
	<groupId>org.apache.kafka</groupId>
	<artifactId>kafka-clients</artifactId>
	<version>3.0.0</version>
</dependency>
```

+ 创建生产者

```java
public static void main(String[] args) throws InterruptedException{
    // 1. 创建kafka生产者的配置对象
	Properties properties = new Properties();

	// bootstrap.servers连接kafka集群配置
    properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,"hadoop102:9092");
    
	// key value序列化配置
    properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
    properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
    
    // 2.创建kafka生产者对象
    KafkaProducer<String,String> kafkaProducer = new KafkaProducer<String,String>(properties);
    
    // 3.调用send方法，发送消息
    for(int i=0; i<5; i++){
        
        // 异步发送
        kafkaProducer.send(new ProducerRecord<>("first","flash7k " + i));
        // 同步发送
        kafkaProducer.send(new ProducerRecord<>("first","flash7k " + i)).get();
        // 带回调函数的异步发送
        kafkaProducer.send(new ProducerRecord<>("first","flash7k " + i),new Callback(){
            // 该方法在 Producer 收到 ack 时调用，为异步调用
			@Override
			public void onCompletion(RecordMetadata metadata, Exception exception) {
				if (exception == null) {
					// 没有异常,输出信息到控制台
					System.out.println("主题："+ metadata.topic() + "->" + "分区：" + metadata.partition());
				} else {
					// 出现异常打印
					exception.printStackTrace();
				}
			}
        });
        
        // 延迟一会会看到数据发往不同分区
		Thread.sleep(2);
    }
    
    // 4.关闭资源
    kafkaProducer.close();
}
```

> + 异步发送：`kafkaProducer.send(new ProducerRecord<>("TopicName","内容"));`
> + 同步发送：`kafkaProducer.send(new ProducerRecord<>("TopicName","内容")).get();`
> + 带回调的异步发送：`kafkaProducer.send(new ProducerRecord<>("TopicName","内容",new Callback(){...}));`

### 生产者分区器

+ 指明分区

```java
kafkaProducer.send(new ProducerRecord<>("TopicName",partition_num,"内容"));
```

> 可通过回调函数查看回调信息

+ 不指明分区，但有key值。将 key 的 hash 值与 topic 的 partition 数进行取余得到 partition 值

```java
kafkaProducer.send(new ProducerRecord<>("TopicName","key","内容"));
```

+ 自定义分区器

  + 实现Partitioner接口，重写partition()方法

  ```java
  package com.flash7k.kafka.producer;
  import org.apache.kafka.clients.producer.Partitioner;
  import org.apache.kafka.common.Cluster;
  import java.util.Map;
      /**
      * 1. 实现接口 Partitioner
      * 2. 实现 3 个方法:partition,close,configure
      * 3. 编写 partition 方法,返回分区号
      */
  public class MyPartitioner implements Partitioner {
       /**
       * 返回信息对应的分区
       * @param topic 主题
       * @param key 消息的 key
       * @param keyBytes 消息的 key 序列化后的字节数组
       * @param value 消息的 value
       * @param valueBytes 消息的 value 序列化后的字节数组
       * @param cluster 集群元数据可以查看分区信息
       * @return
       */
       @Override
       public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
           // 获取消息
           String msgValue = value.toString();
           // 创建 partition
           int partition;
           // 判断消息是否包含 flash7k
           if (msgValue.contains("flash7k")){
              partition = 0;
           }else {
              partition = 1;
           }
           // 返回分区号
           return partition;
       }
       // 关闭资源
       @Override
       public void close() {
       }
       // 配置方法
       @Override
       public void configure(Map<String, ?> configs) {
       }
  }
  ```

  + 使用分区器的方法，在生产者的配置中添加分区器参数

  ```java
  // 添加自定义分区器
  properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG,"com.flash7k.kafka.producer.MyPartitioner");
  ```

## 4）提高吞吐量

**调整参数**

+ `batch.size`：批次大小，默认16k
+ `linger.ms`：等待时间，默认0，可修改为5-100ms
+ `compression.type`：压缩方式，可选择snappy
+ `RecordAccumulator`：缓冲区大小，默认32m，可修改为64m

**代码实现**

```java
package com.atguigu.kafka.producer;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import java.util.Properties;
public class CustomProducerParameters {
	public static void main(String[] args) throws InterruptedException {
        // 1. 创建 kafka 生产者的配置对象
		Properties properties = new Properties();
        // 2. 给 kafka 配置对象添加配置信息：bootstrap.servers
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "hadoop102:9092");

        // key,value 序列化（必须）：key.serializer，value.serializer
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                        "org.apache.kafka.common.serialization.StringSerializer");
		properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                       "org.apache.kafka.common.serialization.StringSerializer");
        // batch.size：批次大小，默认 16K
		properties.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        // linger.ms：等待时间，默认 0
		properties.put(ProducerConfig.LINGER_MS_CONFIG, 1);
        // RecordAccumulator：缓冲区大小，默认 32M：buffer.memory
        properties.put(ProducerConfig.BUFFER_MEMORY_CONFIG,33554432);
        // compression.type：压缩，默认 none，可配置值 gzip、snappy、lz4 和 zstd
		properties.put(ProducerConfig.COMPRESSION_TYPE_CONFIG,"snappy");
        // 3. 创建 kafka 生产者对象
        KafkaProducer<String, String> kafkaProducer = new 
        KafkaProducer<String, String>(properties);
        // 4. 调用 send 方法,发送消息
        for (int i = 0; i < 5; i++) {
            kafkaProducer.send(new 
            ProducerRecord<>("first","atguigu " + i));
        }
        // 5. 关闭资源
        kafkaProducer.close();
     }
}
```

## 5）数据可靠性

**数据完全可靠条件：ACK级别设置为-1，分区副本大于等于2，ISR里应答的最小副本数量大于等于2**

> `ISR`：in-sync replica set，指和Leader保持同步的Follower和Leader集合（leader：0，ISR：0,1,2）
>
> + 如果Follower长时间未向Leader发送通信请求或同步数据，则该Follower将被移出ISR。**该时间阈值由`replica.lag.time.max.ms`参数设定，默认30s**。因此不用长时间等待联系不上或已经故障的节点

**代码实现**

```java
// 设置 acks
properties.put(ProducerConfig.ACKS_CONFIG, "all");
// 重试次数 retries，默认是 int 最大值，2147483647
properties.put(ProducerConfig.RETRIES_CONFIG, 3);
```

## 6）数据去重

### 数据传递语义

+ 至少一次：ACK级别设置为-1，分区副本数大于等于2，ISR里应答的最小副本数大于等于2
+ 至多一次：ACK级别设置为0
+ 总结
  + 至少一次：保证数据不丢失，但不能保证数据不重复
  + 至多一次：保证数据不重复，但不能保证数据不丢失
+ 精确一次：对于一些非常重要的信息，如与钱相关，要求数据不能重复也不能丢失，因此Kafka 0.11版本后，引入**幂等性**和**事务**
  + 精确一次=幂等性+至少一次

### 幂等性

**幂等性原理**

+ 幂等性指Producer不论向Broker发送多少次重复数据，Broker端都只会持久化一条，保证了不重复。

+ **数据重复判断标准**：具有`<PID, Partition, SeqNumber>`相同主键的消息提交时，Broker只会持久化一条
  + `PID`：Kafka每次重启都会分配一个新的`PID`
  + `Partition`：表示分区号
  + `Sequence Number`：单调自增
  + 幂等性只能保证的是**在单分区单会话内不重复**
+ **设置参数**：`enable.idempotence`，默认true，改为false则关闭

### 事务

**事务原理**（开启事务，必须开启幂等性）

1. Kafka Producer 向 Transaction Coordinator事务协调器请求`producer id`（幂等性需要）
2. 事务协调器返回`producer id`
3. Kafka Producer 发送消息到 Topic A
4. Kafka Producer 向事务协调器发送commit请求
5. 事务协调器持久化commit请求 `_transaction_state-分区-Leader`（存储事务信息的特殊Topic）
   + 默认有50个分区，每个分区负责一部分事务。事务划分是根据`transactional.id`的`hashcode%50`， 计算出该事务属于哪个分区。该分区Leader副本所在的broker节点即为这个`transactional.id`对应的Transaction Coordinator节点
6. 事务协调器向 Kafka Producer 返回成功信息
7. 事务协调器后台向 Topic A 发送commit请求
8. Topic A 向事务协调器返回成功信息
9. 事务协调器持久化事务成功信息到 `_transaction_state-分区-Leader`

> Producer在使用事务功能前，必须先自定义一个唯一的`transaction.id`。以便即使客户端突然挂掉，重启时也能继续处理未完成的事务

**事务API**

```java
// 1 初始化事务
void initTransactions();
// 2 开启事务
void beginTransaction() throws ProducerFencedException;
// 3 在事务内提交已经消费的偏移量（主要用于消费者）
void sendOffsetsToTransaction(Map<TopicPartition, OffsetAndMetadata> offsets,String consumerGroupId) throws ProducerFencedException;
// 4 提交事务
void commitTransaction() throws ProducerFencedException;
// 5 放弃事务（类似于回滚事务的操作）
void abortTransaction() throws ProducerFencedException;
```

## 7）数据有序、乱序

+ Kafka在1.x版本之前保证数据单分区有序，`max.in.flight.requests.per.connection=1`（不需要考虑是否开启幂等性）

+ Kafka在1.x版本及以后版本保证数据单分区有序

  + 未开启幂等性：`max.in.flight.requests.per.connection=1`

  + 开启幂等性：`max.in.flight.requests.per.connection`小于等于5

    > 原因说明：因为在Kafka 1.x以后，启用幂等性后，Kafka服务端会缓存producer发来的最近5个request的元数据， 故无论如何，都可以保证最近5个request的数据都是有序的

# 4. Broker

## 1）Broker工作流程

详情见尚硅谷笔记中流程图，很重要！！！

+ Broker启动后在zookeeper中注册（`/brokers/ids/ [0,1,2]`）
+ 先注册的Broker成为Controller，zookeeper记录此信息
+ 由选举出来的 Controller 监听 zookeeper 中 Brokers 节点变化（`/brokers/ids/ [0,1,2]`）
+ Controller决定Leader选举
  + 选举规则：在ISR中存活为前提，按照AR中排在前面的优先。如AR[1,0,2]，ISR[0,1,2]，则Leader按照1,0,2的顺序轮轮询
+ Controller 将节点信息上传到 zookeeper （`/brokers/topics/topic_name/partitions/0/state  "leader:0","isr":[0,1,2]`）
+ 其他 Controller 从zookeeper 同步相关节点信息（以便原 Controller 故障时替换）
+ 假设Broker1中Leader挂了
  + Controller 监听到zk节点变化，从zk获取ISR
  + 选举新的Leader（在ISR中存活为前提，按照AR中排在前面的优先）
  + 更新zk中Leader及ISR信息

## 2）Broker重要参数

| 参数名称                                | 描述                                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| replica.lag.time.max.ms                 | ISR 中，如果 Follower 长时间未向 Leader 发送通信请求或同步数据，则该 Follower 将被踢出 ISR。该时间阈值，默认30s |
| auto.leader.rebalance.enable            | 默认是 true。自动Leader  Partition 平衡                      |
| leader.imbalance.per.broker.percentage  | 默认是 10%。每个 broker 允许的不平衡的 leader的比率。如果每个 broker 超过了这个值，控制器会触发 leader 的平衡 |
| leader.imbalance.check.interval.seconds | 默认值 300 秒。检查 leader 负载是否平衡的间隔时间            |
| log.segment.bytes                       | Kafka 中 log 日志是分成一块块存储的，此配置是指 log 日志划分成块的大小，默认值 1G |
| log.index.interval.bytes                | 默认 4kb，kafka 里面每当写入了4kb大小的日志（.log），然后就往 index 文件里面记录一个索引 |
| log.retention.hours                     | Kafka 中数据保存的时间，默认 7 天                            |
| log.retention.minutes                   | Kafka 中数据保存的时间，分钟级别，默认关闭                   |
| log.retention.ms                        | Kafka 中数据保存的时间，毫秒级别，默认关闭                   |
| log.retention.check.interval.ms         | 检查数据是否保存超时的间隔，默认是 5 分钟。                  |
| log.retention.bytes                     | 默认等于-1，表示无穷大。超过设置的所有日志总大小，删除最早的 segment |
| log.cleanup.policy                      | 默认是 delete，表示所有数据启用删除策略；  如果设置值为 compact，表示所有数据启用压缩策略 |
| num.io.threads                          | 默认是 8。负责写磁盘的线程数。整个参数值要占总核数的 50%     |
| num.replica.fetchers                    | 副本拉取线程数，这个参数占总核数的 50%的 1/3                 |
| num.network.threads                     | 默认是 3。数据传输线程数，这个参数占总核数的 50%的 2/3       |
| log.flush.interval.messages             | 强制页缓存刷写到磁盘的条数，默认是  long 的最大值，9223372036854775807。一般不建议修改，交给系统自己管理 |
| log.flush.interval.ms                   | 每隔多久，刷数据到磁盘，默认是 null。一般不建议修改，交给系统自己管理 |

## 3）节点服役与退役

**服役新节点**

+ 启动原有Kafka集群，单独启动新节点Kafka

+ 执行负载均衡操作：`bin/kafka-reassign-partitions.sh`

  + 创建一个要均衡的主题

  ```json
  vim topics-to-move.json
  
  {
      "topics": [
  		{"topic": "first"}
      ],
      "version": 1
  }
  ```

  + 生成负载均衡计划

  ```shell
  bin/kafka-reassign-partitions.sh -- bootstrap-server hadoop102:9092 --topics-to-move-json-file topics-to-move.json --broker-list "0,1,2,3" --generate
  
  Current partition replica assignment
  {"version":1,"partitions":[{"topic":"first","partition":0,"replic as":[0,2,1],"log_dirs":["any","any","any"]},{"topic":"first","partition":1,"replicas":[2,1,0],"log_dirs":["any","any","any"]},{"topic":"first","partition":2,"replicas":[1,0,2],"log_dirs":["any"," any","any"]}]}
  
  Proposed partition reassignment configuration
  {"version":1,"partitions":[{"topic":"first","partition":0,"replic as":[2,3,0],"log_dirs":["any","any","any"]},{"topic":"first","partition":1,"replicas":[3,0,1],"log_dirs":["any","any","any"]},{"topic":"first","partition":2,"replicas":[0,1,2],"log_dirs":["any"," any","any"]}]}
  
  ```

  + 创建副本存储计划

  ```json
  vim increase-replication-factor.json
  
  {"version":1,"partitions":[{"topic":"first","partition":0,"replic as":[2,3,0],"log_dirs":["any","any","any"]},{"topic":"first","partition":1,"replicas":[3,0,1],"log_dirs":["any","any","any"]},{"topic":"first","partition":2,"replicas":[0,1,2],"log_dirs":["any"," any","any"]}]}
  ```

  + 执行副本存储计划

  ```shell
  bin/kafka-reassign-partitions.sh --bootstrap-server hadoop102:9092 --reassignment-json-file increase-replication-factor.json --execute
  ```

  + 验证：`--verify`

**退役旧节点**

+ 流程与服役新节点一致，区别在于生成负载均衡计划时`--broker-list`去掉旧节点，然后创建副本存储计划，执行
+ 停止Kafka

## 4）Kafka副本策略

### 基本信息

+ 副本作用：提高数据可靠性
+ 副本数量：默认1个，生产环境一般配置2个，保证数据可靠性。副本过多会增加磁盘存储空间，增加网络上数据传输，降低效率
+ 副本类型：Leader、Follower。生产者和消费者只和Leader进行数据传输，Follower找Leader进行同步
+ Kafka 分区中的所有副本统称为 AR（Assigned Repllicas）
  + AR = ISR + OCR
  + ISR：表示和 Leader 保持同步的 Follower 集合。如果 Follower 长时间未向 Leader 发送通信请求或同步数据，则该 Follower 将被踢出 ISR。该时间阈值由`replica.lag.time.max.ms`参数设定，默认 30s。Leader 发生故障之后，就会从ISR 中选举新的Leader
  + OSR：表示Follower 与Leader 副本同步时，延迟过多的副本

### Leader和Follower故障处理

**Follower故障处理细节**

> LEO（Log End Offset）：每个副本的最后一个offset，LEO其实就是最新的offset + 1
>
> HW（High Watermark）：所有副本中最小的LEO

+ Follower故障后会被临时踢出ISR
+ 期间Leader和Follower继续接收数据
+ 待该Follower恢复后，Follower会读取本地磁盘记录的上次的HW，并将log文件高于HW的部分截取掉，从HW开始向Leader进行同步
+ 等该Follower的LEO大于等于该Partition的HW，即Follower追上Leader之后，就可以重新加入ISR了

**Leader故障处理细节**

+ Leader发生故障之后，会从ISR中选出一个新的Leader
+ 为保证多个副本之间的数据一致性，其余的Follower会先将各自的log文件高于HW的部分截掉，然后从新的Leader同步数据
+ **注意：这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复**

## 5）分区副本分配

如果分区数大于服务器台数，如何存储副本？

+ 创建16个分区，3个副本

```shell
bin/kafka-topics.sh --bootstrap-server hadoop102:9092 --create --partitions 16 --replication-factor 3 --topic second
```

+ 查看分区和副本情况

```shell
bin/kafka-topics.sh --bootstrap-server hadoop102:9092 --describe --topic second

Topic:	second4	Partition:	0	Leader:	0	Replicas:	0,1,2 Isr:	0,1,2
Topic:	second4	Partition:	1	Leader:	1	Replicas:	1,2,3 Isr:	1,2,3
Topic:	second4	Partition:	2	Leader:	2	Replicas:	2,3,0 Isr:	2,3,0
Topic:	second4	Partition:	3	Leader:	3	Replicas:	3,0,1 Isr:	3,0,1

Topic:	second4	Partition:	4	Leader:	0	Replicas:	0,2,3 Isr:	0,2,3
Topic:	second4	Partition:	5	Leader:	1	Replicas:	1,3,0 Isr:	1,3,0
Topic:	second4	Partition:	6	Leader:	2	Replicas:	2,0,1 Isr:	2,0,1
Topic:	second4	Partition:	7	Leader:	3	Replicas:	3,1,2 Isr:	3,1,2

Topic:	second4	Partition:	8	Leader:	0	Replicas:	0,3,1 Isr:	0,3,1
Topic:	second4	Partition:	9	Leader:	1	Replicas:	1,0,2 Isr:	1,0,2
Topic:	second4	Partition:	10	Leader:	2	Replicas:	2,1,3 Isr:	2,1,3
Topic:	second4	Partition:	11	Leader:	3	Replicas:	3,2,0 Isr:	3,2,0

Topic:	second4	Partition:	12	Leader:	0	Replicas:	0,1,2 Isr:	0,1,2
Topic:	second4	Partition:	13	Leader:	1	Replicas:	1,2,3 Isr:	1,2,3
Topic:	second4	Partition:	14	Leader:	2	Replicas:	2,3,0 Isr:	2,3,0
Topic:	second4	Partition:	15	Leader:	3	Replicas:	3,0,1 Isr:	3,0,1
```

> 规律：0123,1230,2301,3012,0123
>
> 使Leader尽量均匀分配在各个Broker上

### 手动调整分区副本存储

在生产环境中，每台服务器的配置和性能不一致，但是Kafka只会根据自己的代码规则创建对应的分区副本，就会导致个别服务器存储压力较大。所有需要手动调整分区副本的存储

+ 需求：创建一个新的topic，4个分区，两个副本，名称为three。将该topic的所有副本都存储到broker0和broker1两台服务器上
+ 创建新的topic

```shell
bin/kafka-topics.sh --bootstrap-server hadoop102:9092 --create --partitions 4 --replication-factor 2 --topic three
```

+ 查看分区副本存储情况

```shell
bin/kafka-topics.sh --bootstrap-server hadoop102:9092 --describe --topic three
```

+ **创建副本存储计划（指定副本存储在broker0、broker1中）**

```json
vim increase-replication-factor.json

{
    "version":1, 
    "partitions":[{"topic":"three","partition":0,"replicas":[0,1]},
   				{"topic":"three","partition":1,"replicas":[0,1]},
    			{"topic":"three","partition":2,"replicas":[1,0]},
    			{"topic":"three","partition":3,"replicas":[1,0]}]
}
```

+ 执行副本存储计划

```shell
bin/kafka-reassign-partitions.sh -- bootstrap-server hadoop102:9092 --reassignment-json-file increase-replication-factor.json --execute
```

+ 验证：`--verify`、查看：`--describe`

### Leader Partitioner负载均衡

正常情况下，Kafka本身会自动把Leader  Partition均匀分散在各个机器上，来保证每台机器的读写吞吐量都是均匀的。但是如果某些broker宕机，会导致Leader Partition过于集中在其他少部分几台broker上，这会导致少数几台broker的读写请求压力过高，其他宕机的broker重启之后都是Follower partition，读写请求很低，造成集群负载不均衡

**参数设置**

| 参数名称                                  | 描述                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| `auto.leader.rebalance.enable`            | 默认是 true。 自动Leader  Partition 平衡。生产环境中，Leader 重选举的代价比较大，可能会带来性能影响，建议设置为 false  关闭 |
| `leader.imbalance.per.broker.percentage`  | 默认是 10%。每个 broker 允许的不平衡的 Leader 的比率。如果每个 broker 超过了这个值，控制器会触发 Leader 的平衡 |
| `leader.imbalance.check.interval.seconds` | 默认值 300 秒。检查 Leader 负载是否平衡的间隔时间            |

案例

```linux
Topic:	flash7k	Partition: 0 Leader: 0 Replicas: 3,0,2,1 Isr: 3,0,2,1
Topic:	flash7k	Partition: 1 Leader: 1 Replicas: 1,2,3,0 Isr: 1,2,3,0
Topic:	flash7k	Partition: 2 Leader: 2 Replicas: 0,3,1,2 Isr: 0,3,1,2
Topic:	flash7k	Partition: 3 Leader: 3 Replicas: 2,1,0,3 Isr: 2,1,0,3
```

> + 针对broker0节点，分区2的AR优先副本是0节点，但是0节点却不是Leader节点，所以不平衡数加1，AR副本总数是4。broker0节点不平衡率为1/4>10%，需要再平衡
>
> + broker2和broker3节点和broker0不平衡率一样，需要再平衡
>
> + Broker1的不平衡数为0，不需要再平衡

### 增加副本因子

在生产环境当中，由于某个主题的重要等级需要提升，我们考虑增加副本。副本数的增加需要先制定计划，然后根据计划执行

+ 创建topic（假设有三个分区，只有一个副本）
+ 手动增加副本存储（即三个分区所有副本都指定存储到broker0、broker1、broker2上

```json
{	"version":1,
 	"partitions":[{"topic":"four","partition":0,"replicas":[0,1,2]},
                  {"topic":"four","partition":1,"replicas":[0,1,2]},
                  {"topic":"four","partition":2,"replicas":[0,1,2]}
                 ]
}
```

+ 执行副本存储计划

## 6）文件存储

### 文件存储机制

+ Topic：逻辑上的概念

+ Partition：物理上的概念
  + 每个Partition对应于一个log文件，该log文件中存储的就是生产者生产的数据。
  + 生产者生产的数据会被不断**追加**到该log文件末端。为防止log文件过大导致数据定位效率低下，Kafka采取了**分片**和**索引**机制。
  + 每个log文件分为多个segment。每个segment包括：`.index`、`.log`、`.timeindex`等文件。这些文件位于一个文件夹下，该文件夹的命名规则为：topic名称+分区序号，例如：first-0
    + `.index`：偏移量索引文件
    + `.log`：日志文件，也就是数据文件
    + `.timeindex`：时间戳索引文件
    +  说明：`.index`和`.log`文件以当前segment的第一条消息的offset命名

> 可通过工具查看`.index`和`.log`文件
>
> ```shell
> cd /opt/module/kafka/datas/first-1
> 
> kafka-run-class.sh kafka.tools.DumpLogSegments --files ./00000000000000000000.index
> ```

#### Log文件和Index文件详解：稀疏索引

如何在log文件中定位到offset=600的Record？

1. 根据目标offset**定位Segment**文件
2. 找到小于等于目标offset的最大offset对应的**index索引项**
3. **定位到log文件**
4. **向下遍历找到目标Record**

> 注意：
>
> + index为稀疏索引。大约每往log中写入4kb数据，就会往index中写入一条索引。`log.index.interval.bytes` 默认4kb
> + index文件中保存的是相对offset（即在这个segment中的相对位置），确保offset的值所占空间不会过大，能控制在固定大小

**日志存储参数**

| 参数                       | 描述                                                         |
| -------------------------- | ------------------------------------------------------------ |
| `log.segment.bytes`        | Kafka 中 log 日志是分成一块块存储的，此配置是指 log 日志划分成块的大小，默认值 1G |
| `log.index.interval.bytes` | 默认 4kb，kafka 里面每当写入了 4kb 大小的日志（.log），然后就往 index  文件里面记录一个索引。 稀疏索引 |

## 7）文件清理策略

Kafka 中默认的日志保存时间为 7 天，可以通过调整如下参数修改保存时间。

+ `log.retention.hours`：最低优先级小时，默认 7 天，设为3天即可
+ `log.retention.minutes`：分钟
+ `log.retention.ms`：最高优先级毫秒
+ `log.retention.check.interval.ms`：负责设置检查周期，默认 5 分钟

日志一旦超过了设置的时间，如何处理？

### 日志清除策略：Delete、Compact

**Delete**：`log.cleanup.policy = delete`，删除过期数据

+ 基于时间：默认打开。以 segment 中所有记录中的最大时间戳作为该文件时间戳
+ 基于大小：默认关闭。超过设置的所有日志总大小，删除最早的 segment。`log.retention.bytes`默认等于-1，表示无穷大

如果一个segment中有一部分过期，另一部分没有过期，如何处理？

**Compact**：`log.cleanup.policy = delete`，压缩日志数据，即对应相同key的不同value值，只保留最后一个版本

+ 压缩后的offset可能是不连续的。当从这些offset消费消息时，将会拿到比这个offset大的offset对应的消息，并从这个位置开始消费。
+ Compact策略只适合特殊场景。比如消息的key是用户ID，value是用户的资料，通过Compact，整个消息集里就保存了所有用户最新的资料。

## 8）高效读写数据

+ 分布式集群，再采用分区技术，提高并行度
+ 读数据采用稀疏索引，可以快速定位要消费的数据
+ 顺序写磁盘：写数据的过程是一直追加到文件末端，不是随机写，省去大量磁头寻址时间
+ 页缓存+零拷贝技术

### 页缓存+零拷贝

**页缓存**：Kafka重度依赖底层操作系统提供的PageCache功能

+ 当上层有写操作时， 操作系统只是将数据写入PageCache
+ 当读操作发生时，先从PageCache中查找。如果找不到，再去磁盘中读取
+ 实际上PageCache是把尽可能多的空闲内存都当做了磁盘缓存来使用

**零拷贝**：Kafka的数据加工处理操作交由Kafka生产者和Kafka消费者处理。Kafka Broker应用层不关心存储的数据，所以就不用走应用层，传输效率高

**参数**

| 参数名称                      | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| `log.flush.interval.messages` | 强制页缓存刷写到磁盘的条数， 默认是 long 的最大值，9223372036854775807。一般不建议修改，交给系统自己管理 |
| `log.flush.interval.ms`       | 每隔多久，刷数据到磁盘，默认是 null。一般不建修改，交给系统自己管理 |

# 5. 消费者

## 1）消费方式

+ Pull（拉取）模式：
  + Consumer采用从broker中主动拉取数据。**Kafka采用这种方式**
  + 不足：如果Kafka没有数据，消费者可能会陷入循环中，一直返回空数据
+ Push（推）模式：
  + 由broker 决定消息发送速率，很难适应所有消费者的消费速率

## 2）消费者工作流程



## 3）消费者组原理

Consumer Group（CG）：消费者组，由多个 Consumer 组成

形成一个消费者组的条件：所有消费者的 Groupid 相同。

+ 消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费

  + 如果向消费组中添加更多的消费者，超过主题分区数量，则有一部分消费者就会闲

    置，不会接收任何消息

+ 消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者

### 消费者组初始化流程

**Coordinator**：辅助实现消费者组的初始化和分区的分配

+ `coordinator`节点选择 = `groupid`的`hashcode`值 % 50（ `consumer_offsets`的分区数量）
+ 例如： `groupid`的`hashcode`值 = 1，1% 50 = 1，那么` consumer_offsets `主题的1号分区在哪个broker上，就选择这个节点的`coordinator`作为这个消费者组的老大。消费者组下的所有的消费者提交offset的时候就往这个分区去提交offset

1. 每个 Consumer 都发出 Join Group 请求
2. Coordinator 选出一个Consumer作为Leader
3. Coordinator 把要消费的topic情况发送给Consumer Leader
4. Consumer Leader 制定消费方案
5. Consumer Leader 把消费方案发给 Coordinator 
6. Coordinator 把这个消费方案发送给各个 Consumer 
7. 每个消费者都会和 Coordinator保持心跳（默认3s）
   + 一旦超时（`session.timeout.ms=45s`），该消费者会被移除，并触发再平衡
   + 消费者处理消息的时间过长（`max.poll.interval.ms=5mins`），也触发再平衡

### 消费者组详细消费流程

## 4）消费者组重要参数

| 参数名称                                   | 描述                                                         |
| ------------------------------------------ | ------------------------------------------------------------ |
| `bootstrap.servers`                        | 向 Kafka  集群建立初始连接用到的 host/port 列表              |
| `key.deserializer`和  `value.deserializer` | 指定接收消息的 key 和  value  的反序列化类型。一定要写全类名 |
| `group.id`                                 | 标记消费者所属的消费者组                                     |
| `enable.auto.commit`                       | 默认值为 true，消费者会自动周期性地向服务器提交偏移量        |
| `auto.commit.interval.ms`                  | 如果设置了 `enable.auto.commit`  的值为 true， 则该值定义了消费者偏移量向Kafka 提交的频率，默认 5s |
| `auto.offset.reset`                        | 当 Kafka 中没有初始偏移量或当前偏移量在服务器中不存在  （如，数据被删除了），该如何处理？ `earliest`：自动重置偏移量到最早的偏移量。  latest：默认，自动重置偏移量为最新的偏移量。` none`：如果消费组原来的（previous）偏移量不存在，则向消费者抛异常。<br />`anything`：向消费者抛异常 |
| `offsets.topic.num.partitions`             | `consumer_offsets` 的分区数，默认是 50 个分区                |
| `heartbeat.interval.ms`                    | Kafka 消费者和 coordinator  之间的心跳时间，默认 3s。  该条目的值必须小于session.timeout.ms ，也不应该高于  session.timeout.ms 的 1/3 |
| `session.timeout.ms`                       | Kafka 消费者和 coordinator 之间连接超时时间，默认45s。超过该值，该消费者被移除，消费者组执行再平衡 |
| `max.poll.interval.ms`                     | 消费者处理消息的最大时长，默认是 5 分钟。超过该值，该消费者被移除，消费者组执行再平衡 |
| `fetch.min.bytes`                          | 默认 1 个字节。消费者获取服务器端一批消息最小的字节数        |
| `fetch.max.wait.ms`                        | 默认 500ms。如果没有从服务器端获取到一批数据的最小字节数。该时间到，仍然会返回数据 |
| `fetch.max.bytes`                          | 默认Default: 52428800（50 m）。消费者获取服务器端一批消息最大的字节数。如果服务器端一批次的数据大于该值  （50m）仍然可以拉取回来这批数据，因此，这不是一个绝对最大值。一批次的大小受  `message.max.bytes `（ broker  config）or `max.message.bytes `（topic config）影响 |

## 5）消费者API

### 独立消费者（订阅主题/分区）

> **注意：**在消费者 API 代码中必须配置消费者组 id。命令行启动消费者不填写消费者组
>
> id 会被自动填写随机的消费者组 id。

```java
package com.flash7k.kafka.consumer;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import java.time.Duration;
import java.util.ArrayList;
import java.util.Properties;
public class CustomConsumer {
    public static void main(String[] args) {
		// 0. 创建消费者的配置对象
        Properties properties = new Properties();
        // 给消费者配置对象添加参数
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, 
        "hadoop102:9092");
        // 配置序列化（必配）
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
        StringDeserializer.class.getName());
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
        StringDeserializer.class.getName());
        // 配置消费者组（组名任意起名）（必配）
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "test");
        
        // 1. 创建消费者对象
        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<String, String>(properties);
        
        // 2.1 订阅要消费的主题（可以消费多个主题）
        ArrayList<String> topics = new ArrayList<>();
        topics.add("first");
        kafkaConsumer.subscribe(topics);
        
        // 2.2 订阅某个主题的某个分区数据
		ArrayList<TopicPartition> topicPartitions = new ArrayList<>();
 		topicPartitions.add(new TopicPartition("first", 0));
 		kafkaConsumer.assign(topicPartitions);
        
        // 3. 消费数据
        while (true) {
            // 设置 1s 中消费一批数据
            ConsumerRecords<String, String> consumerRecords = 
            kafkaConsumer.poll(Duration.ofSeconds(1));
            // 打印消费到的数据
            for (ConsumerRecord<String, String> consumerRecord : consumerRecords) {
            	System.out.println(consumerRecord);
            }
        }
    }
}
```

### 消费者组

## 6）分区的分配及再平衡

| 参数名称                      | 描述                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| heartbeat.interval.ms         | Kafka 消费者和 Coordinator之间的心跳时间，默认 3s。该条目的值必须小于 session.timeout.ms ，  也不应该高于  session.timeout.ms 的 1/3 |
| session.timeout.ms            | Kafka 消费者和 Coordinator之间连接超时时间，默认 45s。超过该值，该消费者被移除，消费者组执行再平衡 |
| max.poll.interval.ms          | 消费者处理消息的最大时长，默认是 5 分钟。超过该值，该消费者被移除，消费者组执行再平衡 |
| partition.assignment.strategy | 消 费者分区分配策略 ，默认策略是Range + CooperativeSticky。Kafka 可以同时使用多个分区分配策略。可以选择的策略包括： Range 、 RoundRobin 、 Sticky、CooperativeSticky |

### Range

**Range 是对每个 Topic 而言的**

+ 首先对同一个 Topic 里面的 Partitions 按照序号进行排序，并对 Consumer 按照字母顺序进行排序
  + 假如现在有 7 个分区，3 个消费者，排序后的分区将会是0,1,2,3,4,5,6；消费者排序完之后将会是 C0,C1,C2
+ 通过 Partitions 数/ Consumer 数来决定每个消费者应该消费几个分区。**如果除不尽，那么前面几个消费者将会多消费 1 个分区**
  + 例如，7/3 = 2 余 1 ，除不尽，那么 消费者 C0 便会多消费 1 个分区。 8/3=2余2，除不尽，那么C0和C1分别多消费一个
  + 结果：C0 - 0，1，2 、C1 - 3，4 、C2 - 5，6
  + 再平衡：C1 - 0，1，2，3 、C2 - 4，5，6

> 注意：如果只是针对 1 个 Topic 而言，C0 消费者多消费 1 个分区影响不是很大。但是如果有 N 多个 Topic ，那么针对每个  Topic ，消费者 C0都将多消费 1 个分区， Topic 越多，C0消费的分区会比其他消费者明显多消费N 个分区。
>
> **容易产生数据倾斜！**

### RoundRobin

**RoundRobin 针对集群中所有 Topic 而言**

+ 首先把所有的 Partitions 和所有的 Consumer 都列出来
+ 然后按照 hashcode 进行排序
+ 最后通过**轮询**算法来分配 Partitions  给到各个 Consumer 
  + 结果：C0 - 0，3，6 、C1 - 1，4 、C2 - 2，5
  + 再平衡：C1 - 0，2，4，6 、C2 - 1，3，5

**修改方法**

```java
// 修改分区分配策略
properties.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,"org.apache.kafka.clients.consumer.RoundRobinAssignor");
```

### Sticky

**粘性分区定义**：可以理解为分配的结果带有“粘性的”。即在执行一次新的分配之前，
考虑上一次分配的结果，尽量少的调整分配的变动，可以节省大量的开销。

+ 粘性分区是 Kafka 从 0.11.x 版本开始引入这种分配策略
+ 首先会**尽量均衡的随机放置**分区到消费者上面
+ 在同一消费者组内消费者出现问题的时候，会尽量保持原有分配的分区不变化
  + 结果：C0 - 0，1 、C1 - 2，3，5 、C2 - 4，6
  + 再平衡：C1 - 2，3，5 、C2 - 0，1，4，6

**修改方法**

```java
// 修改分区分配策略
ArrayList<String> startegys = new ArrayList<>();
startegys.add("org.apache.kafka.clients.consumer.StickyAssignor");

properties.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,startegys);
```

## 7）Offset偏移量

**默认维护位置**

+ Kafka 0.9版本之前，Consumer 默认将 offset 保存在Zookeeper中
+ 从0.9版本开始，Consumer 默认将 offset 保存在 Kafka 一个内置的 Topic 中，该 Topic 为`__consumer_offsets`
  + `__consumer_offsets` 主题里面采用 key 和 value 的方式存储数据
  + key 是 group.id+topic+分区号
  + value 是当前 offset 的值
  + 每隔一段时间，kafka 内部会对这个 topic 进行compact，也就是每个 group.id+topic+分区号就保留最新数据

**消费offset**

+ 在配置文件 `config/consumer.properties` 中添加配置 `exclude.internal.topics=false`，
  默认是 true，表示不能消费系统主题。为了查看该系统主题数据，所以该参数修改为 false。
+ 采用命令行方式，创建一个新的 Topic

```shell
bin/kafka-topics.sh --bootstrap-server hadoop102:9092 --create --topic flash7k --partitions 2 --replication-factor 2
```

+ 启动生产者往 flash7k生产数据

```shell
bin/kafka-console-producer.sh --topic flash7k --bootstrap-server hadoop102:9092
```

+ 启动消费者往 flash7k生产数据

```shell
bin/kafka-console-consumer.sh --topic flash7k --bootstrap-server hadoop102:9092 --group test
```

> 注意：指定消费者组名称，更好观察数据存储位置（key 是 group.id+topic+分区号）

+ 消费`__consumer_offsets`

```shell
bin/kafka-console-consumer.sh --topic __consumer_offsets --bootstrap-server hadoop102:9092 --consumer.config config/consumer.properties --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" --from-beginning


[offset,flash7k,1]::OffsetAndMetadata(offset=7,leaderEpoch=Optional[0],metadata=,commitTimestamp=1622442520203,expireTimestamp=None)
[offset,flash7k,0]::OffsetAndMetadata(offset=8,leaderEpoch=Optional[0],metadata=,commitTimestamp=1622442520203,expireTimestamp=None)
```

### 自动提交offset

| 参数名称                  | 描述                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `enable.auto.commit`      | 默认值为 true，消费者会自动周期性地向服务器提交偏移量        |
| `auto.commit.interval.ms` | 如果设置了 `enable.auto.commit `的值为 true， 则该值定义了消费者偏移量向 Kafka 提交的频率，默认 5s |

**实现：添加配置即可**

```java
// 是否自动提交 offset
properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG,true);
// 提交 offset 的时间周期 1000ms，默认 5s
properties.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG,1000);
```

### 手动提交offset

虽然自动提交offset十分简单便利，但由于其是基于时间提交的，开发人员**难以把握offset提交的时机**。因此Kafka还提供了手动提交offset的API。

+ `commitSync`（同步提交）、`commitAsync`（异步提交）
+ 相同点：都会将本次提交的一批数据最高的偏移量提交
+ 不同点：
  + 同步提交阻塞当前线程，一直到提交成功，并且会自动失败重试（由不可控因素导致，也会出现提交失败）
  + 异步提交则没有失败重试机制，故有可能提交失败
  +  `commitSync`：必须等待offset提交完毕，再去消费下一批数据
  +  `commitAsync` ：发送完提交offset请求后，就开始消费下一批数据了。

**`commitSync`（同步提交）**

有失败重试机制，故更加可靠，但是由于一直等待提交结果，提交的效率比较低

+ 配置

```java
// 是否自动提交 offset
properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG,false);
```

+ 消费时

```java
// 同步提交 offset
consumer.commitSync();
```

**`commitAsync`（异步提交）**

虽然同步提交 offset 更可靠一些，但是由于其会阻塞当前线程，直到提交成功。因此吞吐量会受到很大的影响。更多的情况下，会选用异步提交 offset的方式。

+ 配置

```java
// 是否自动提交 offset
properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG,false);
```

+ 消费时

```java
// 异步提交 offset
consumer.commitAsync();
```

### 指定offset消费

```properties
auto.offset.reset = earliest | latest | none ##默认是 latest
```

当 Kafka 中没有初始偏移量（消费者组第一次消费）或服务器上不再存在当前偏移量
时（例如该数据已被删除），该怎么办？

+ `earliest`：自动将偏移量重置为**最早**的偏移量。相当于--from-beginning
+ `latest`（默认值）：自动将偏移量重置为**最新**偏移量
+ `none`：如果未找到消费者组的先前偏移量，则向消费者抛出异常

**代码实现**

```java
package com.atguigu.kafka.consumer;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.ArrayList;
import java.util.HashSet;
import java.util.Properties;
import java.util.Set;
public class CustomConsumerSeek {
    public static void main(String[] args) {
        // 0 配置信息
        Properties properties = new Properties();
        // 连接
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,"hadoop102:9092");
        // key value 反序列化
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,StringDeserializer.class.getName());
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,StringDeserializer.class.getName());
        // 创建GroupId
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "test2");
        
        // 1 创建一个消费者
        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(properties);
        // 2 订阅一个主题
        ArrayList<String> topics = new ArrayList<>();
        topics.add("first");
        kafkaConsumer.subscribe(topics);
        
        // 3 指定offset
        Set<TopicPartition> assignment= new HashSet<>();
        while (assignment.size() == 0) {
            // 消费者拉取数据，强迫生成分区分配信息
            kafkaConsumer.poll(Duration.ofSeconds(1));
            // 获取消费者分区分配信息（有了分区分配信息才能开始消费）
            assignment = kafkaConsumer.assignment();
        }
        // 遍历所有分区，并指定 offset 从 1700 的位置开始消费
        for (TopicPartition tp: assignment) {
        	kafkaConsumer.seek(tp, 1700);
        }
        
        
        // 4 消费该主题数据
        while (true) {
            ConsumerRecords<String, String> consumerRecords = kafkaConsumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> consumerRecord :consumerRecords) {
            	System.out.println(consumerRecord);
            }
        }
    }
}
```

#### 指定时间消费

在以上代码中修改以下部分

```java
// 2 订阅一个主题
ArrayList<String> topics = new ArrayList<>();
topics.add("first");
kafkaConsumer.subscribe(topics);

// 拉取数据获得分区信息
Set<TopicPartition> assignment = new HashSet<>();
while (assignment.size() == 0) {
    kafkaConsumer.poll(Duration.ofSeconds(1));
    // 获取消费者分区分配信息（有了分区分配信息才能开始消费）
    assignment = kafkaConsumer.assignment();
}

// 封装集合存储，每个分区对应一天前的数据
HashMap<TopicPartition, Long> timestampToSearch = new HashMap<>();
for (TopicPartition topicPartition : assignment) {
    timestampToSearch.put(topicPartition,System.currentTimeMillis() - 1 * 24 * 3600 * 1000);
}
// 获取从 1 天前开始消费的每个分区的 offset
Map<TopicPartition, OffsetAndTimestamp> offsets = kafkaConsumer.offsetsForTimes(timestampToSearch);
// 遍历每个分区，对每个分区设置消费时间。
for (TopicPartition topicPartition : assignment) {
    OffsetAndTimestamp offsetAndTimestamp = offsets.get(topicPartition);
    // 根据时间指定开始消费的位置
    if (offsetAndTimestamp != null){
        kafkaConsumer.seek(topicPartition, offsetAndTimestamp.offset());
    }
}

// 3 消费数据
```

+ **重复消费**：已经消费了数据，但是 offset没提交
  + 自动提交offset引起：每5s提交一次，如果在提交后的5s内消费者挂了，再次启动消费者使，则需从上次提交的offset处继续消费，导致重复消费
+ **漏消费**：先提交 offset后消费，有可能会造成数据的漏消费
  + 手动提交offset引起：当offset被提交时，数据还在内存中未落盘，此时消费者挂了，offset已经提交，但是数据还未消费，造成漏消费
+ **解决办法：消费者事务**

## 8）消费者事务

Kafka消费端将消费过程和提交offset过程做原子绑定。此时需要将Kafka的offset保存到支持事务的自定义介质，如MySQL。（待学）

## 9）消费者如何提高吞吐量

+ 如果是**Kafka消费能力不足**：考虑增加Topic的分区数，并且同时提升消费组的消费者数量，使消费者数 = 分区数。（两者缺一不可）
+ 如果是下游的**数据处理不及时**：提高每批次拉取的数量。批次拉取数据过少（拉取数据/处理时间 < 生产速度），使处理的数据小于生产的数据，也会造成数据积压

| 参数名称           | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| `fetch.max.bytes`  | 默认 Default: 52428800（50 m）。消费者获取服务器端一批消息最大的字节数。如果服务器端一批次的数据大于该值（50m）仍然可以拉取回来这批数据，因此，这不是一个绝对最大值。一批次的大小受 `message.max.bytes` （broker config）or `max.message.bytes `（topic config）影响 |
| `max.poll.records` | 一次 poll拉取数据返回消息的最大条数，默认是 500 条           |

# 6. Kafka-Eagle 监控

# 7. Kafka-Kraft 模式



