# 浅谈Kafka核心原理

####  简介

​	Kafka起初是由LinkedIn公司采用Scala语言开发的一个多分区、多副本且基于Zookeeper协调的分布式消息系统，现已捐献给Apache基金会。目前Kafka已经被定为为一个分布式流式处理平台，它以高吞吐、可持久化、可水平拓展、支持流数据处理等多种特性而被广泛使用。

​	Kafka之所以受到越来越多的青睐，与它所”扮演“的三大角色是分不开的：

​	**消息系统：**作为消息中间件，具备系统解耦、冗余存储、流量削峰、缓冲、异步通信、扩展性、可恢复性等功能。与此同时，Kafka还提供了大多数消息系统难以实现的消息顺序性保障及回溯消费等功能。

​	**存储系统：**Kafka把消息持久化到磁盘，有效降低了数据丢失的风险。也正是得益于Kafka的消息持久化功能和多副本机制，我们可以把Kafka作为长期的数据存储系统来使用，只需要把对应的数据保留策略设置为“永久”或启用主题的日志压缩功能即可。

​	**流式处理平台：**

#### 消息队列（Message Queue）使用场景

##### ·  解耦：

###### 未使用MQ的耦合场景：

​	现有A服务在自己代码中调用B服务的接口和C服务的接口发送数据

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570349327919.png)

​		此时新增D服务也需要A服务发送数据，则需要A服务在自己代码里修改，发送数据给D服务，紧接着C服务又说不需要A服务给自己发送数据了

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570353961427.png)

​		负责A服务的人还得考虑，如果调用的B服务挂了怎么办？如果D服务访问超时怎么办？由于A服务产生了比较关键的数据，许多服务需要A服务发送该数据过来，这也导致了A服务与其他服务的严重耦合。

**使用MQ解耦场景：**

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570354023234.png)

我们自己使用的场景

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1571154041749.png)

##### ·  异步：

**未使用MQ的同步高延时请求场景：**

​	现有一用户请求，调用服务A接口

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570355189888.png)

​		我们来计算一下，服务A先是在自己本地执行SQL，然后调用了服务B、服务C和服务D的接口，4个步骤下来，需要耗时的总时长为970ms。用户通过浏览器发起请求，等待1秒才得到响应，几乎不可接受。一般对于用户的直接的操作，要求是每个请求都必须在200ms内完成，对用户几乎是无感知的。

**使用MQ进行异步化：**

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570358228532.png)

​	使用MQ进行异步化之后，此时用户发起请求调用服务A的总耗时变成了20+5=25ms。 

##### ·  削峰：

**未使用MQ削峰大量用户请求场景：**

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570376006119.png)

**使用MQ进行削峰场景：**

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570377103963.png)

​		MQ中每秒有2000个请求进来，就只有1000个请求出去，结果就是导致在高峰期（假设1个小时）可能有几十万甚至上百万的请求积压在MQ中，但是高峰期过后，每秒钟只有20个请求，系统还是会按照每秒1000个请求的速度处理，差不多1个多小时就可以把积压的上百万条消息给处理掉，就没有积压了。



#### 引入MQ后可能存在的一些问题

**系统可用性降低：**系统引入的外部依赖越多，越容易挂掉。拿上图举例，MQ一旦故障，A服务没发发送消息到MQ了，然后BCD服务也没发消费到消息了，整个系统就崩溃了，没法运转了。

**系统复杂性提高：**消息丢失，消息重复，消息顺序性问题如何保证？例如A服务本来只需要给B服务发送一条数据就可以了，结果因为A服务和MQ之间协调出现问题，A服务不小心把同一条数据发了两次到MQ中给B服务消费，导致B服务插入两条一模一样的数据。

**一致性问题：**如A服务处理完了直接返回成功了，都认为这个请求成功了，但是要BCD服务都写库成功才是真正的成功，如果其中有一个写库失败了，这样数据就不一致了。



#### 典型的Kafka体系架构

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570434633144.png)

先简单介绍下Kafka中的术语：

（1）Producer：生产者，也就是发送消息的一方。生产者负责创建消息，然后将其投递到Kafka中。

（2）Consumer：消费者，也就是接收消息的一方。消费者连接到Kafka上并接收消息，进而进行相应的业务逻辑处理。

（3）Broker：服务代理节点。可以将其看做一台服务器上部署的一台Kafka服务器，前提是这台服务器上只部署了一个Kafka实例。一个或多个Broker组成了一个Kafka集群。

（4）Topic：主题。Kafka中的消息以主题为单位进行归类，生产者负责将消息发送到特定的主题，而消费者负责订阅主题并进行消费。

（5）Partition：分区。一个topic可以分为多个partition，每个partition是一个有序的队列。

（6）offset：偏移量。同一个topic下的不同partition包含的消息是不同的，partition在存储层面可以看作一个可追加的日志文件，消息在被追加到分区日志的时候都会分配一个特定的偏移量（offset）。offset是消息在分区中的唯一标识，Kafka通过它来保证消息在分区中的顺序性，不过offset并不跨越分区，也就是说，Kafka保证的是分区有序而不是主题有序。

如图，某个主题中有3个分区，消息被顺序追加到每个分区日志文件的尾部。Kafka中的分区可以分布在不同的broker上，也就是说，一个topic的数据可以分布在多个broker上

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570435400123.png)

​	Kafka之所以将topic分成多个分区，分布在不同的broker上，就是提供负载均衡的能力，也就是实现系统的高伸缩性。

#### Kafka多副本机制

​		Kafka为分区引入了多副本（Replica）机制，通过增加副本数量可以提升容灾能力。同一分区的不同副本中保存的是相同的消息（在同一时刻，副本之间并非完全一样），副本之间是“一主多从”的关系，其中leader副本负责处理读写请求，follower副本只负责与leader副本的消息同步。副本处于不同的broker中，当leader副本出现故障时，从follower副本中从新选举新的leader副本对外提供服务。Kafka通过多副本机制实现 了故障的自动转移，当Kafka集群中某个新的broker失效时，仍然能保证服务可用。

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570458914836.png)

​		如图所示，Kafka集群中有3个broker，某个topic中有3个分区，且副本因子（即副本个数）也为3，如此每个分区便有1个leader副本和2个follower副本。生产者和消费者只与leader副本进行交互，而follower副本只负责消息的同步，很多时候follower副本中的消息相对于leader而言会有一定的滞后。

​		分区中的所有副本统称为 **AR**（Assigned Replicas）。所有与leader副本保持一定程度的同步的副本（包括leader副本在内）组成 **ISR**（In-Sync Replicas），ISR集合是AR集合的一个子集。消息会先发送到leader副本，然后follower副本才能从leader副本中拉取消息进行同步，同步期间内follower副本相对于leader副本会有一定程度的滞后。与leader副本同步滞后过多的副本（不包括leader副本）组成**OSR**（Out-of-Sync Replicas），由此可见，AR = ISR + OSR。在正常情况下，所有的follower副本都应该与leader副本保持一定程度的同步，即 AR = ISR，OSR集合为空。

​		leader副本负责维护和跟着ISR集合中所有follower副本的滞后状态，当follower副本落后太多或失效时，leader副本会把它从ISR集合中剔除。如果OSR集合中所有的follower副本“追上”了leader副本，那么leader副本会把它从OSR集合转移至ISR集合。默认情况下，当leader副本发生故障时，只有在ISR集合中的副本才有资格被选举为新的leader。

​		ISR与HW和LEO也有紧密的关系。HW是High Watermark的缩写，俗称高水位，它标识了一个特定的消息偏移量（offset），消息只能拉取到这个offset之前的消息。

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570463237141.png)

​		如上图所示，表示一个分区中各种偏移量的说明。它代表一个日志文件，这个日志文件中有9条消息，第一条消息的offset为0，最后一条消息的offset为8,offset为9代表下一条待写入的消息的位置。日志文件的HW为6，表示消费者只能拉取到offset在0至5之间的消息，而offset为6的消息对消费者而言是不可见的。**LEO**是Log End Offset的缩写，标识当前日志文件下一条待写入的消息的offset。分区ISR集合中的每个副本都会维护自身的的LEO，而集合中最小的LEO即为分区的HW，对消费者而言，只能消费HW之前的消息。下面举个例子来更好的说明ISR集合与HW和LEO之间的关系：

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570466677914.png)

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570466932355.png)

​	在同步过程中，不同的follower副本的同步效率也不尽相同。

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570467298346.png)

​		在某一时刻，follower1完全跟上了leader副本而follower2只同步到了消息3，如此leader副本的LEO为5，follower1的LEO为5，follower2的LEO为4，那么当前分区的HW取最小值4，此时消费者可以消费到offset为0至3之间的消息。

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570467513662.png)

​		所有的消息都成功写入了消息3和消息4，整个分区的HW和LEO都变为5，因此消费者可以消费到offset为4的消息了。由此可见，Kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制。事实上，同步复制要求所有能工作的follower副本都复制完，这条消息才会被确认为已成功提交，这种复制方式极大地影响了性能。而在异步复制方式下，follower副本异步地从leader副本中复制数据，数据只要被leader副本写入就被认为已经成功提交了。在这种情况下，如果follower副本都还没有复制完而落后于leader副本，突然leader副本宕机，则会造成数据丢失。Kafka使用的这种ISR的方式则有效地权衡了数据可靠性和性能之间的关系。

#### 生产者

​	一个正常的生产逻辑为以下几个步骤：配置客户端参数，创建相应的生产者实例，构建待发送的消息，发送消息。

​	客户端参数配置

```java
	Properties props = new Properties();
	props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, brokerList);
	props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
	props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
```

​		bootstrap.servers：该参数用来指定生产者客户端连接Kafka集群所需的broker地址清单，格式为host1:port1,host2:port2,这里不一定需要配置所以的broker地址，因为生产者会从给定的broker里找到其他broker信息。但至少配置2个以上，当其中一个宕机了，能够保证生产者仍然能连接到kafka集群上。key.serializer和value.serializer指定序列化操作的序列化器。这三个参数都没有默认值，所以在配置生产者客户端时是必填的。

```java
properties.put("acks","0");
```

​		acks，可取值0，1，-1，这个参数是用来指定分区中必须要有多少个副本接收到这条消息，之后生产者才会认为这条消息写入成功。默认值为1，生产者发送消息之后，只要分区的leader副本成功写入消息，那么它就会收到来自服务端的成功响应。如果消息写入leader副本并返回成功响应给生产者，且在被其他follower副本拉取前leader副本崩溃了，那么此时消息还是会丢失，因为新选举的leader副本中并没有这条对应的消息。acks=0，生产者发送消息之后不需要等待任何服务端的响应。这样可以达到最大的吞吐量，但是也存在问题，如果在消息发送到写入Kafka的过程中出现某些异常，导致Kafka没有接收到这条消息，那么生产者也不知道，消息也就丢失了。acks=-1或acks=all，生产者在消息发送之后，需要等待ISR中的所有副本都成功写入消息之后才能够收到来自服务端的成功响应。设置成-1可以达到最强的可靠性，但这并不意味着消息就一定可靠，因为如果ISR中可能只有leader副本，这样就退化成acks=1的情况了。所以acks默认为1，是消息可靠性和吞吐量之间的一个折中方案。

​	构建消息，即创建ProducerRecord对象。

```java
public class ProducerRecord<K,V> {
    private final String topic; //主题
    private final Integer partition; //分区号
    private final K key; //键
    private final V value; //值
    private final Long timestamp; //消息的时间戳
    ...
}
```

​		其中topic和partition字段分别指代消息要发往的主题和分区号。value是指消息体，即你要发送的内容。key是用来指定消息的键，它不仅是消息的附加信息，还可以用来计算分区号进而可以让消息发往特定的分区。消息以主题为单位进行归类，而这个key可以让消息再进行二次归类，同一个key的消息会被划分到同一个分区中。说到key，这里如果要保证消息的顺序性，可以把需要保证消息消费顺序的指定同一个key。消息在通过send()方法发往broker的过程中，有可能需要经过拦截器、序列化器和分区器。拦截器一般不是必需的，但序列化器是必需的。生产者需要用序列化器把对象转换成字节数组才能通过网络发送给Kafka。

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570549324737.png)

​		如果在构造消息时在ProducerRecord中指定了partition字段，那么就不需要分区器的作用，如果没有指定，那么就需要依赖分区器根据key这个字段来计算partition的值。在默认分区器的方法中，如果key部位null，那么默认的分区器会对key进行哈希，最终根据等到的哈希值来计算分区号，有相同key的消息会被写入同一个分区。如果key为null，那么消息将会以轮询的方式发往主题内的各个可用分区。

#### 消费者

**消费者**(Consumer)：负责订阅Kafka中的主题（topic），并且从订阅的主题上拉取消息。

**消费组**（Consumer Group）：每个消费者都有一个对应的消费组，消息发布到主题后，只会被投递给订阅它的每个消费组中的一个消费者。

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570634727822.png)

​		如上图所示，某个主题共有3个分区，有两个消费组A和B都订阅了这个主题。按照Kafka默认的规则，消费组A中每个消费者分配到1个分区，消费组B中C3分配到两个分区，C4分配到1个分区。两个消费组之间互不影响，每个消费组只能消费所分配到的分区中的消息，换言之，每一个分区只能被一个消费组中的一个消费者所消费。消费组是一个逻辑上的概念，它将属于同一组的消费者归为一类，每一个消费者只隶属于一个消费组，课通过消费者客户端参数group.id来配置消费组。

对于消息中间件一般有两种消息投递模式：

**点对点（P2P，Point-to-Point）模式**：基于队列，生产者发送消息到队列，消费者从队列中接收消息。一般是一对一。

**发布/订阅（Pub/Sub）模式**：主题作为消息传递的中介，生产者将消息发布到某个主题，消费者可主题中订阅消息。该模式在消息的一对多广播时采用。

​	Kafka同时支持两种消息投递模式，而这正得益于它的消费者与消费组模型设计：

- 如果所有的消费者都隶属于同一个消费组，那么所有的消息都会被均匀的投递给每一个消费者，即每条消息只会被一个消费者处理，这就相当于点对点模式。
- 如果所有的消费者都隶属于不同的消费组，那么所有的消息都会被广播给所有的消费者，即每条消息会被所有的消费者处理，这就相当于发布/订阅模式。



​		一个正常的**消费逻辑步骤**：配置消费者客户端参数，创建消费者实例，订阅主题，拉取消息并消费，提交消费位移等。

​		配置必要的消费者客户端参数，有4个参数是必填的。同生产者一样，bootstrap.servers、key.deserializer和value.deserializer三个参数是必配的，只不过key、value的变成了反序列化器，还有一个group.id配置消费者隶属的消费组名称。

```java
props.put(ConsumerConfig.GROUP_ID_CONFIG, "goupA");
```

​		消息的消费一般有两种模式：推模式——是服务端主动将消息推送给消费者，拉模式——是消费者主动向服务端发起请求来拉取消息。Kafka中的消费基于拉模式的。Kafka中的消息消费是一个不断轮询的过程，消费者所要做的就是重复地调用poll（）方法，返回所订阅的主题（分区）上的一组消息。

​	消费者消费到的每条消息类型为ConsumerRecord，这个和生产者发送的消息类型ProducerRecord对应，不过字段更丰富：

```java
public class ConsumerRecord<K,V> {
    private final String topic; //消息所属主题名称
    private final int partition; //消息所属分区编号
    private final long offset; //消息所属分区偏移量
    private final long timestamp; //时间戳
    private final TimestampType timestampType; //两种类型，CreateTime消息创建的时间戳，												   //LogAppendTime消息追加到日志的时间戳
    private final K key;
    private final V value; //一般业务应用所要读取的值
    ...
}
```

**位移提交**

​		Kafka中每条消息都有唯一的offset，表示该消息处在的partition中的位置，叫作“偏移量”。消费者中也有一个offset概念，表示消费者消费到分区中某个消息所在的位置，我们把它与消息的区分开，可叫作“位移”。在旧消费者客户端（用Scala编写的客户端版本）中，消费位移是保存在ZooKeeper中的，而在新消费者客户端（用Java编写的客户端）中，消费位移存储在Kafka内部的主题_consumer_offsets中。这里将消费位移存储起来（持久化）的动作称为“提交”。

![](https://raw.githubusercontent.com/ly8051033/BlogPicture/master/pic/1570640588699.png)

​		当前消费者消费的位移为X，但它需要提交的消费位移不是X，而是X+1，它表示下一条需要拉取的消息的位置。在Kafka中默认的消费位移提交方式是自动提交，提交时间默认为5秒，可通过auto.commit.interval.ms配置。

​		自动提交位移的方式非常简便，但是也会带来重复消费和消息丢失的问题。

​		假设刚刚提交完一次消费位移，然后拉取一批消息进行消费，在下一次自动位移提交之前，消费者崩了，那么等消费者恢复再来消费消息的时候又得从上一次位移提交的地方重新开始，这样便发生了重复消费的现象。我们可以通过减小位移提交时间间隔来减小重复消息的窗口，但这样并不能避免重复消费的发送，而且也会使得位移提交更加频繁。这里我们可以在拿数据写库前，根据主键去库中查询，如果已有，就update一下好了，若是写入redis，用set存储，去重。



@by 罗云	

