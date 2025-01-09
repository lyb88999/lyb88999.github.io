# Kafka学习笔记

## Kafka简要介绍

定位：消息队列

作用：削峰、解耦、异步通信

特点：高吞吐、低延迟

为什么使用消息队列？举一个生活中的例子

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141235605.png)

再看业务中的问题，对于一个下单功能

1. 如果是同步的流程的话：

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141300765.png)

会有下面几个问题：

**问题一：耦合度高**

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141316720.png)

**问题二：响应时间长**

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141401331.png)

**问题三：并发压力传递**

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141419504.png)

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141438678.png)

**问题四：系统结构弹性不足**

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141452755.png)

2. 改成异步流程：

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141507632.png)

**好处一：功能解耦**

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141519794.png)

**好处二：快速响应**

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640.png)



**好处三：异步削峰限流**

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141537854.png)

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141550305.png)

**好处四：系统结构弹性大，易于扩展**

![图片](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/640-20241127141603075.png)

## Kafka基础架构

### 基础架构图

![img](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/2603267-20230517110625281-781547790.jpg)

1. **Producer**：消息生产者，就是向`Kafka broker`发消息的客户端。
2. **Consumer**：消息消费者，向`Kafka broker`取消息的客户端，多个`Consumer`会组成一个消费者组。
3. **Broker**：`Kafka`的服务端由被称为`Broker`的服务进程构成，即一个`Kafka`集群由多个`Broker`组成，`Broker`负责接收和处理客户端发送过来的请求，以及对消息进行持久化。
4. **Topic** ：可以理解为一个队列，一个`Kafka集群`中可以定义很多的`topic`，比如上图中的`topicA`。
5. **Partition**：为了实现扩展性，提高吞吐量，一个非常大的`topic`可以分布到多个`broker`上，也就是一个`topic`可以分为多个`partition`，每个`partition`是一个有序的队列，比如上面的`topicA`被分为了三个`partition`：`partition0`、`partition1`和`partition2`，分别分布在`broker0`、`broker1`和`broker2`上。每个分区是一组有序的消息日志。多个`partition`也可以放在一个`broker`上。
6. **Replica**：副本，如果数据只放在一个`broker`中，万一这个`broker`宕机了怎么办？为了实现高可用，一个`topic`的每个分区都有若干个副本，一个`Leader`和若干个`Follower`，比如上图中每个`partition`连接的就是它的副本，`replica`分布在与`partition`不同的`broker`中。
7. **Leader**：每个分区的多个副本的“主”，生产者发送数据的对象，以及消费者消费数据的对象都是`Leader`，比如上图中红色旗子标志的分区。
8. **Follower**：每个分区中多个副本的“从”，实时从`Leader`中同步数据。`Leader`发生故障时，某个`Follower`会成为新的`Leader`，比如上图中黑色旗子标志的分区。
9. **Zookeeper**：用来记录`Kafka`中的一些元数据，比如`kafka`集群中的`broker`，`leader`是谁等等，但`Kafka` 2.8.0之后也支持非zk的方式，大大减少了和zk的交互。

与**ES**有同样思想的地方：

1. 分片机制：两者都采用分片机制来进行数据分布，Kafka是叫Partition，ES是叫Replica；分片都设置副本来提供高可用性，都叫Replica。
2. 选举机制：都采用类似的Leader-Follower模型，当Leader节点故障时，会触发新的Leader的选举，不过这里有个不一样的点是Kafka只有Leader分区提供读写服务，Follower只提供备份，而ES支持从Follower读取数据。

## Kafka生产者流程

在消息的发送过程中，涉及到了两个线程——`main`线程和`Sender`线程。在`main`线程中创建了一个双端队列`RecordAccumulator`。`main`线程将消息发送给`RecordAccumulator`，`Sender`线程不断从`RecordAccumulator`中拉取消息发送到`Kafka broker`。

### Kafka架构图

![img](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/2603267-20230517110625206-2137213608.jpg)

1. 在`main`线程中由`Kafka producer`创建信息，然后通过可能的拦截器`interceptors`、序列化器`serializer`和分区器`partitioner`的作用之后缓存到消息累加器`RecordAccumulator`中。
   * 拦截器：可以用来在消息发送前做一些准备工作，比如按照某个规则过滤不符合要求的信息、修改消息的内容等，也可以用来在发送回调逻辑前做一些定制化的需求，比如统计类工作。
   * 序列化器：用于在网络传输中将数据序列化为字节流进行传输，保证数据不丢失。
   * 分区器：用于按照一定规则将数据分发到不同的`Kafka broker`节点中。
2. `Sender`线程负责从`RecordAccumulator`获取消息并将其发送到`Kafka`中。
   * `RecordAccumulator`主要用来缓存消息以便`Sender`线程可以批量发送，进而减少网络传输的资源消耗以提高性能。
   * `RecordAccumulator`缓存的大小可以通过生产者客户端参数`buffer.memory`配置，默认值为`33554432B`，即`32MB`。
   * 主线程中发来的消息都会被加到`RecordAccumulator`的某个双端队列中，`RecordAccumulator`内部为每个分区都维护了一个双端队列，即`Deque<ProducerBatch>`，消息写入缓存时，追加到双端队列的尾部。
   * `Sender`读取消息时，从双端队列的头部读取。`ProducerBatch`是指一个消息批次；与此同时，会将较小的`ProducerBatch`凑成一个较大的`ProducerBatch`，也可以减少网络请求的次数以提升整体的吞吐量。`ProducerBatch`的大小可以通过`batch.size`控制，默认为`16k`。
   * `Sender`线程会在有数据积累到`batch.size`，或者如果数据迟迟未达到`batch.size`，`Sender`线程等待`linger.ms`设置的时间到了之后会获取数据。`linger.ms`单位为`ms`，默认是`0ms`，表示没有延迟。
3. `Sender`从`RecordAccumulator`获取缓存的消息后，会将数据封装成网络请求`<Node, Request>`的形式，这样就可以将`Request`请求发送给各个`Node`了。
4. 请求在从`Sender`线程发送到`Kafka`之前还会保存到`InflightRequests`中，它的主要作用是缓存了已经发出去但还没有收到服务器响应的请求。`InflightRequests`默认每个分区下最多缓存5个请求，通过`max.in.flight.request.per.connection`修改。
5. 请求`Request`通过通道`Selector`发送到`Kafka`节点。
6. 发送后，需要等待`Kafka`的应答机制，取决于配置项`acks`（默认值是-1）。
   * `0`：发送者发送过来的数据，不需要等待数据落盘就应答。
   * `1`：发送者发送过来的数据，`Leader`收到数据后应答。
   * `-1`：发送者发送过来的数据，`Leader`和副本节点收齐数据后应答。
7. `Request`请求接收到`Kafka`响应的结果，如果成功的话，从`InflightRequests`清除请求，否则的话需要重发操作，可以通过配置项`retries`决定，当消息发送出现错误的时候，系统会重发消息，`retries`表示重试次数，默认是int最大值。
8. 清理消息累加器`RecordAccumulator`中的数据。

## Kafka消费者流程

`Kafka`中的消费者是基于拉取模式的。消息的消费一般有两种模式：推送模式和拉取模式。推模式是服务端主动将消息推送给消费者，而拉模式是消费者主动向服务端发起请求来拉取消息。

Kafka是以消费组形式进行消费的，一个消费者组，由多个`consumer`组成。形成一个消费者组的条件，是所有消费者的`groupid`相同。

![img](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/2603267-20230517110625226-187208903.png)

* 消费者组内每个消费者负责消费不同分区的数据，一个分区只能由一个组内消费者消费。如果向消费者组中添加更多的消费者，超过主题分区数量，则有一部分消费者就会闲置，不会接收任何消息。（对于同一个消费者组，消费者和分片是一对多，分片和消费者是一对一）[一个分区只能由一个组内消费者消费保证了一个消息在`group`内只消费一次；不同的消费者组消费同一个`topic`不受影响]
  * 分片和消费者一对一：`Kafka`中的每个`Partition`是一个有序且不变的消息队列，这样可以保证`Partition`的消息被按顺序消费；当系统需要更高的消费能力时，可以通过增加`Partition`的数量来水平扩展，每个新增的`Partition`都可以被不同的消费者处理，增加整体的消费能力。
  * 消费者和分片一对多：允许一个消费者同时消费多个`Partition`，可以充分利用消费者的处理能力，提高整体的消费吞吐量；且当消费者组中的消费者增加或减少时，`Kafka`可以重新分配`Partition`到不同的消费者，实现消费者的动态伸缩。

* 消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。

那么现在的问题是：`Kafka`是如何指定消费者组的每个消费者消费哪个分区，每次消费的数量是多少呢？

1. 制定消费方案

   ![img](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/2603267-20230517110625216-1455636030.jpg)

   1. 消费者`ConsumerA`、`ConsumerB`、`ConsumerC`向`Kafka`集群中的协调器`coordinator`发送`JoinGroup`的请求，`coordinator`主要是来辅助实现消费者组的初始化和分区的分配。
      * `coordinator`老大的选择=`groupid`的`hashcode`值%50（`__consumer_offsets`内置主题位移的分区数量），例如`groupid`的值为1，1%50=1，那么`__consumer_offsets`主题的一号分区，在哪个`broker`上，就选择这个节点的`coordinator`作为这个消费者组的老大，消费者组下所有的消费者提交`offset`的时候就往这个分区去提交`offset`。(`__consumer_offsets`是`Kafka`自己创建的一个`topic`，用来保存`consumer`提交的位移)
   2. 选出一个`consumer`作为消费中的`leader`，比如上图中的`ConsumerB`。
   3. 消费者`leader`制定出消费方案，比如谁来消费哪个分区等。
   4. 把消费方案发送给`coordinator`。
   5. 最后`coordinator`就把消费方案下发给各个`consumer`。

   注意，每个消费者都会和`coordinator`保持心跳（默认3s），一旦超时（`session.timeout.ms=45s`），该消费者会被移除，并触发再平衡；或者消费者处理消息的时间过长（`max.poll.interval.ms`=5分钟），也会触发再平衡，也就是重新进行上面的流程。

2. 消费者消费细节

   ![img](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/2603267-20230517110625345-1627998639.jpg)

   1. 消费者创建一个网络连接客户端`ConsumerNetworkClient`，发送消费请求，可以进行如下配置：
      - `fetch.min.bytes`: 每批次最小抓取大小，默认1字节
      - `fetch.max.bytes`: 每批次最大抓取大小，默认50M
      - `fetch.max.wait.ms`：最大超时时间，默认500ms
   2. 发送请求到`Kafka`集群。
   3. 成功的回调，会把数据保存到`completedFetches`队列中。
   4. 消费者从队列中抓取数据，根据配置`max.poll.records`一次拉取数据返回消息的最大条数，默认500条。
   5. 获取到数据后，需要经过反序列化器、拦截器等。

## Kafka安装部署

* Docker-compose 部署

```yam
version: "3"
services:
  zookeeper:
    image: 'bitnami/zookeeper:latest'
    ports:
      - '2181:2181'
    environment:
      # 匿名登录--必须开启
      - ALLOW_ANONYMOUS_LOGIN=yes
    volumes:
      - ./zookeeper:/bitnami/zookeeper
  # 该镜像具体配置参考 https://github.com/bitnami/bitnami-docker-kafka/blob/master/README.md
  kafka:
    image: 'bitnami/kafka:2.8.0'
    ports:
      - '9092:9092'
      - '9999:9999'
    environment:
      - KAFKA_BROKER_ID=1
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092
      # 客户端访问地址，更换成自己的
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://47.93.190.244:9092
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      # 允许使用PLAINTEXT协议(镜像中默认为关闭,需要手动开启)
      - ALLOW_PLAINTEXT_LISTENER=yes
      # 开启自动创建 topic 功能便于测试
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=true
      # 全局消息过期时间 6 小时(测试时可以设置短一点)
      - KAFKA_CFG_LOG_RETENTION_HOURS=6
      # 开启JMX监控
      - JMX_PORT=9999
    volumes:
      - ./kafka:/bitnami/kafka
    depends_on:
      - zookeeper
  # Web 管理界面 另外也可以用exporter+prometheus+grafana的方式来监控 https://github.com/danielqsj/kafka_exporter
  kafka_manager:
    image: 'hlebalbau/kafka-manager:latest'
    ports:
      - "9000:9000"
    environment:
      ZK_HOSTS: "zookeeper:2181"
      APPLICATION_SECRET: letmein
    depends_on:
      - zookeeper
      - kafka
```

然后执行docker-compose up命令即可

## cli操作Kafka

启动Kafka容器后，用docker exec -it kafka-kafka-1 /bin/bash命令进入kafka容器的bash环境

![image-20241125212116209](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241125212116209.png)

新建一个名为StandAlone的topic

![image-20241126113233885](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241126113233885.png)

然后启动生产者，输入kafka-console-producer.sh --bootstrap-server localhost:9092 --topic StandAlone

![image-20241125212419370](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241125212419370.png)

此时就可以往Kafka里面发送消息了

现在再启动一个消费者来订阅该topic，kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic StandAlone --from-beginning，如果不加--from-beginning参数会默认从当前时间开始消费StandAlone topic中的新消息，加了以后就是从topic的起始位置开始消费所有消息

![image-20241125212956973](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241125212956973.png)

如果此时再用--from-beginning新启动一个消费者，那么就还是会从头开始读Kafka对应topic中的数据

![image-20241125213132793](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241125213132793.png)

但如果不用--from-beginning参数的话，就会默认从当前时间开始消费

还可以指定--partition xxx --offset earliest等参数设置，第一个参数表示如果topic有多个分区的话，可以通过制定分区来消费，第二个参数表示从分区的最早位置开始消费，还可以是latest，也就是从分区的最新位置开始消费，当然也可以是具体的offset值，直接从该offset值位置开始消费

还可以用--group xxx指定消费者组

## Kafka的Golang库——Sarama

Kafka有五类核心API：

* `Producer API`：应用程序通过这套API和向主题发送消息
* `Consumer API`：应用程序通过这套API订阅一个或多个主题，并从所订阅的消息中拉取消息
* `Streams API`：应用程序通过这套API实现流处理器，可以将一个主题的消息“导流”到另一个主题，并能对消息进行任意自定义的转换
* `Connector API`：应用程序通过这套API来实现连接器，这些连接器不断地从源系统或应用程序导入数据到Kafka，反过来也可将Kafka消息不断地导入某个接收系统或应用程序
* `Admin API`：应用程序通过这套API管理和检查主题、Broker和其他Kafka实体

`Kafka`中异步生产者和同步生产者的主要区别如下：

1. 处理方式不同：同步生产者在发送消息时会阻塞等待响应；而异步生产者发送消息不会阻塞
2. 性能差异：同步生产者每条消息都要等待服务器响应，吞吐量较低，适合对可靠性较高的场景；异步生产者不用等待服务器响应，吞吐量高，适合对性能要求高的场景，可以批量发送消息
3. 错误处理：同步生产者直接通过返回值处理错误；异步生产者需要单独的goroutine处理错误

**Producer**

1. 同步生产者

```go
func Produce(cfg *conf.Config, topic string, limit int) error {
	if cfg == nil {
		return fmt.Errorf("config is nil")
	}
	config := sarama.NewConfig()
	config.Producer.Return.Successes = true
	config.Producer.Return.Errors = true
	// 同步生产者和异步生产者的逻辑是一致的，Success或者Error都是通过channel返回的
	// 只是同步生产者封装了一层，等channel返回之后才返回给调用者
	producer, err := sarama.NewSyncProducer([]string{cfg.Host}, config)
	if err != nil {
		return fmt.Errorf("failed to new sync producer: %w", err)
	}
	defer func() {
		if err := producer.Close(); err != nil {
			log.Println("failed to close producer: ", err)
		}
	}()
	var successes, errors int
	for i := 0; i < limit; i++ {
		str := strconv.Itoa(int(time.Now().UnixNano()))
		msg := &sarama.ProducerMessage{Topic: topic, Key: nil, Value: sarama.StringEncoder(str)}
		// SendMessage函数会返回生产的消息所在的分片partition和位移offset，如果有错误会返回error
		partition, offset, err := producer.SendMessage(msg)
		if err != nil {
			log.Println("failed to send message ", err)
			errors++
			continue
		}
		successes++
		log.Printf("[Producer] partitionid: %d; offset: %d; value: %s\n", partition, offset, str)
	}
	log.Printf("发送完毕 总发送条数:%d successes: %d errors: %d\n", limit, successes, errors)
	return err
}
```

测试：

```go
var cfg *conf.Config

func init() {
	var err error
	cfg, err = conf.LoadConfig("../../../.")
	if err != nil {
		log.Fatalln("failed to load config: ", err)
	}
}

func main() {
	// 生产的topic为cfg.Topic
	sync.Produce(cfg, "standAlone", 1000)
}
```

日志：

![image-20241121145028404](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241121145028404.png)

2. 异步生产者

```go
func Producer(ctx context.Context, cfg *conf.Config, topic string, limit int) error {
	if cfg == nil {
		return fmt.Errorf("config is nil")
	}

	config := sarama.NewConfig()
	config.Producer.Return.Errors = true
	config.Producer.Return.Successes = false // 不需要返回成功确认
	config.Producer.Partitioner = sarama.NewRandomPartitioner

	producer, err := sarama.NewAsyncProducer([]string{cfg.Host}, config)
	if err != nil {
		return fmt.Errorf("create async producer failed: %w", err)
	}

	var (
		wg                        sync.WaitGroup
		enqueued, timeout, errors atomic.Int64
	)

	// 处理错误
	wg.Add(1)
	go func() {
		defer wg.Done()
		for e := range producer.Errors() {
			errors.Add(1)
			log.Printf("[Producer] Error: err:%v msg:%+v \n", e.Msg, e.Err)
		}
	}()

	// 异步发送消息
	for i := 0; i < limit; i++ {
		select {
		case <-ctx.Done():
			log.Printf("context cancelled, stop sending messages")
			goto END
		default:
			str := strconv.FormatInt(time.Now().UnixNano(), 10)
			msg := &sarama.ProducerMessage{
				Topic: topic,
				Value: sarama.StringEncoder(str),
			}

			// 使用固定超时时间的context
			sendCtx, cancel := context.WithTimeout(ctx, time.Millisecond*100)
			select {
			case producer.Input() <- msg:
				enqueued.Add(1)
			case <-sendCtx.Done():
				timeout.Add(1)
			}
			cancel()

			if i > 0 && i%100 == 0 {
				log.Printf("已发送消息数:%d 超时数:%d\n",
					enqueued.Load(), timeout.Load())
			}
		}
	}

END:
	producer.AsyncClose()
	wg.Wait()

	log.Printf("发送完毕 总发送条数:%d enqueued:%d timeout:%d errors:%d\n",
		limit,
		enqueued.Load(),
		timeout.Load(),
		errors.Load(),
	)
	return nil
}
```

测试：

```go
var cfg *conf.Config

func init() {
	var err error
	cfg, err = conf.LoadConfig("../../../.")
	if err != nil {
		log.Fatalln("failed to load config: ", err)
	}
}
func main() {
	// 默认消费cfg.Topic
	async.Producer(context.Background(), cfg, "standAlone", 10000)
}
```

日志：

![image-20241121155453344](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241121155453344.png)

**Consumer**

1. 独立消费者

​	1.1 单分区消费

```go
func SinglePartition(cfg *conf.Config, topic string) error {
	config := sarama.NewConfig()
	consumer, err := sarama.NewConsumer([]string{cfg.Host}, config)
	if err != nil {
		log.Printf("failed to new consumer: %v", err)
		return err
	}
	defer func(consumer sarama.Consumer) {
		err := consumer.Close()
		if err != nil {
			log.Printf("failed to close the consumer: %v", err)
		}
	}(consumer)
	// 参数1 指定哪个topic
	// 参数2 分区 默认消费0号分区
	// 参数3 offset 从哪里起开始消费 offsetNewest offsetOldest分别代表从最新的开始消费和从最初的开始消费
	// 这里demo就从最新的开始消费，即该 consumer 启动之前产生的消息都无法被消费
	partitionConsumer, err := consumer.ConsumePartition(topic, 0, sarama.OffsetNewest)
	if err != nil {
		log.Printf("failed to consume partition: %v", err)
		return err
	}
	defer func(partitionConsumer sarama.PartitionConsumer) {
		err := partitionConsumer.Close()
		if err != nil {
			log.Printf("failed to close the partitionConsumer: %v", err)
		}
	}(partitionConsumer)
	for message := range partitionConsumer.Messages() {
		log.Printf("[Consumer] partitionid: %d; offset:%d, value: %s\n", message.Partition, message.Offset, string(message.Value))
	}
	return nil
}
```

​	测试：

```go
func main() {
    cfg, err := conf.LoadConfig("../../..")
    if err != nil {
       log.Fatalln("failed to load config: ", err)
    }
    topic := cfg.Topic
    standalone.SinglePartition(cfg, topic)
    // standalone.Partitions(cfg, topic)
}
```

​	日志：

![image-20241121161143460](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241121161143460.png)

​	1.2 多分区消费

```go
// Partitions 每次都从第一条消息开始消费 一直到消费完全部消息
func Partitions(cfg *conf.Config, topic string) error {
	config := sarama.NewConfig()
	consumer, err := sarama.NewConsumer([]string{cfg.Host}, config)
	if err != nil {
		log.Printf("failed to new consumer: %v", err)
		return err
	}
	defer func(consumer sarama.Consumer) {
		err = consumer.Close()
		if err != nil {
			log.Printf("failed to close the consumer: %v", err)
		}
	}(consumer)
	
	// 查询该topic有多少个分区
	partitions, err := consumer.Partitions(topic)
	if err != nil {
		log.Printf("failed to get the number of partitions: %v", err)
		return err
	}
	log.Println(len(partitions))

	var wg sync.WaitGroup
	wg.Add(len(partitions))
	for _, partitionID := range partitions {
		go consumeByPartition(consumer, topic, partitionID, &wg)
	}
	wg.Wait()
	return nil
}


func consumeByPartition(consumer sarama.Consumer, topic string, partitionID int32, wg *sync.WaitGroup) error {
	defer wg.Done()
	partitionConsumer, err := consumer.ConsumePartition(topic, partitionID, sarama.OffsetOldest)
	if err != nil {
		log.Printf("failed to consumePartition %d: %v", partitionID, err)
		return err
	}
	defer func(partitionConsumer sarama.PartitionConsumer) {
		err = partitionConsumer.Close()
		if err != nil {
			log.Printf("failed to close partitionConsumer: %v", err)
		}
	}(partitionConsumer)
	for message := range partitionConsumer.Messages() {
		log.Printf("[Consumer] partitionid: %d; offset:%d, value: %s\n", message.Partition, message.Offset, string(message.Value))
	}
	return nil
}
```

​	测试：

```go
func main() {
	cfg, err := conf.LoadConfig("../../..")
	if err != nil {
		log.Println("failed to load config: ", err)
	}
	topic := cfg.TopicPartition
	topic = "testPartition2"
	// standalone.SinglePartition(cfg, topic)
	async.Producer(context.Background(), cfg, topic, 10000)
	err = standalone.Partitions(cfg, topic)
	if err != nil {
		log.Println("failed to partitions consume: ", err)
	}
}
```

​	```日志```

![image-20241121174233778](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241121174233778.png)

反复运行上面的 Demo 会发现，每次都会从第 1 条消息开始消费，一直到消费完全部消息， 因为我们给消费设置的offset是`sarama.OffsetOldest`。

为了防止每次重启消费者都从第 1 条消息开始消费，**我们需要在消费消息后将 offset 提交给 Kafka**。这样重启后就可以接着上次的 Offset 继续消费了。

​	1.3 OffsetManager

```go
func OffsetManager(cfg *conf.Config, topic string) error {
	config := sarama.NewConfig()
	config.Consumer.Offsets.AutoCommit.Enable = true              // 开启自动 commit offset
	config.Consumer.Offsets.AutoCommit.Interval = 1 * time.Second // 自动commit时间间隔
	client, err := sarama.NewClient([]string{cfg.Host}, config)
	if err != nil {
		log.Printf("failed to new client: %v", err)
		return err
	}
	defer func(client sarama.Client) {
		err := client.Close()
		if err != nil {
			log.Printf("failed to close the client: %s", err)
		}
	}(client)
	offsetManager, err := sarama.NewOffsetManagerFromClient("myGroupID", client)
	if err != nil {
		log.Printf("failed to new offset manager from client: %s", err)
		return err
	}
	defer func(offsetManager sarama.OffsetManager) {
		err := offsetManager.Close()
		if err != nil {
			log.Printf("failed to close the offsetManager: %s", err)
		}
	}(offsetManager)
	partitionOffsetManager, err := offsetManager.ManagePartition(topic, cfg.DefaultPartition)
	if err != nil {
		log.Printf("failed to manage partition offset: %s", err)
		return err
	}
	defer func(partitionOffsetManager sarama.PartitionOffsetManager) {
		err := partitionOffsetManager.Close()
		if err != nil {
			log.Printf("failed to close the partitionOffsetManager: %s", err)
		}
	}(partitionOffsetManager)
	// 在程序结束后再commit一次，防止自动提交间隔之间的信息被丢掉
	defer offsetManager.Commit()
	consumer, err := sarama.NewConsumerFromClient(client)
	if err != nil {
		log.Printf("failed to new consumer from client :%s", err)
		return err
	}
	// 根据Kafka中记录的上次消费的offset开始+1的位置接着消费
	nextOffset, _ := partitionOffsetManager.NextOffset()
	log.Println("nextOffset: ", nextOffset)
	pc, err := consumer.ConsumePartition(topic, cfg.DefaultPartition, nextOffset)
	if err != nil {
		log.Printf("ConsumePartition err: %s", err)
		return err
	}
	defer func(pc sarama.PartitionConsumer) {
		err := pc.Close()
		if err != nil {
			log.Printf("failed to close the partition Consumer: %s", err)
		}
	}(pc)
	for message := range pc.Messages() {
		value := string(message.Value)
		log.Printf("[Consumer] partitionid: %d; offset:%d, value: %s\n", message.Partition, message.Offset, value)
		// 每次消费后都更新一次offset，这里更新的只是程序内存中的值，需要commit之后才能提交到kafka
		partitionOffsetManager.MarkOffset(message.Offset+1, "modified metadata")
	}
	return nil
}
```

​	测试：

```go
func main() {
	cfg, err := conf.LoadConfig("../../")
	if err != nil {
		log.Fatalln("failed to load config: ", err)
	}
	topic := cfg.Topic
	offsetmanager.OffsetManager(cfg, topic)
}
```

日志：

![image-20241121182436296](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241121182436296.png)



2. 消费者组

```go
// Kafka消费者组中可以存在多个消费者，Kafka会以Partition为单位将消息分发给各个消费者
// 每条消息只会被消费者组的一个消费者消费

type MyConsumerGroupHandler struct {
	name  string
	count int64
}

// Setup 执行在 获得新 session 后 的第一步, 在 ConsumeClaim() 之前
func (MyConsumerGroupHandler) Setup(_ sarama.ConsumerGroupSession) error {
	return nil
}

// Cleanup 执行在 session 结束前, 当所有 ConsumeClaim goroutines 都退出时
func (MyConsumerGroupHandler) Cleanup(_ sarama.ConsumerGroupSession) error { return nil }

func (h MyConsumerGroupHandler) ConsumeClaim(sess sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	for msg := range claim.Messages() {
		fmt.Printf("[consumer] name:%s topic:%q partition:%d offset:%d\n", h.name, msg.Topic, msg.Partition, msg.Offset)
		// 标记消息已被消费 内部会更新 consumer offset
		sess.MarkMessage(msg, "")
		h.count++
		if h.count%1000 == 0 {
			fmt.Printf("name:%s 消费数:%v\n", h.name, h.count)
		}
	}
	return nil
}

func ConsumerGroup(cfg *conf.Config, topic string, name string) {
	config := sarama.NewConfig()
	config.Consumer.Return.Errors = true
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()
	cg, err := sarama.NewConsumerGroup([]string{cfg.Host}, cfg.ConsumerGroupID, config)
	if err != nil {
		log.Fatal("failed to new consumer group: ", err)
	}
	defer func(cg sarama.ConsumerGroup) {
		err := cg.Close()
		if err != nil {
			log.Fatal("failed to close the consumer group")
		}
	}(cg)
	var wg sync.WaitGroup
	wg.Add(2)
	go func() {
		defer wg.Done()
		for err := range cg.Errors() {
			fmt.Println("ERROR", err)
		}
	}()
	go func() {
		defer wg.Done()
		handler := MyConsumerGroupHandler{name: name}
		for {
			log.Println("running: ", name)
			/*
				![important]
				应该在一个无限循环中不停地调用 Consume()
				因为每次 Rebalance 后需要再次执行 Consume() 来恢复连接
				Consume 开始才发起 Join Group 请求 如果当前消费者加入后成为了 消费者组 leader,则还会进行 Rebalance 过程，重新分配
				组内每个消费组需要消费的 topic 和 partition，最后 Sync Group 后才开始消费
			*/
			err = cg.Consume(ctx, []string{topic}, handler)
			if err != nil {
				log.Println("Consume err: ", err)
			}
			// 如果 context 被 cancel 了，那么退出
			if ctx.Err() != nil {
				return
			}
		}
	}()
	wg.Wait()
}
```

测试：

```go
func main() {
	cfg, err := conf.LoadConfig("../../..")
	if err != nil {
		log.Fatalln("failed to load config: ", err)
	}
	// async.Producer(context.Background(), cfg, "ConsumerGroup", 10000)
	// 测试: 该topic创建时只指定两个分区，然后启动3个消费者进行消费
	// 结果: 最终只有两个消费者能够消费到消息
	go group.ConsumerGroup(cfg, "ConsumerGroup", "C1")
	go group.ConsumerGroup(cfg, "ConsumerGroup", "C2")
	group.ConsumerGroup(cfg, "ConsumerGroup", "C3")
}
```

日志：

![image-20241122105944985](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241122105944985.png)

消费者组中C1和C2分别负责消费分区0和分区1，C3不参与消费，因为同一个topic中一个分区只能被消费者组内一个消费者消费。
