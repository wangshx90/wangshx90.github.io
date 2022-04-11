---
title:  Kafka原理学习之协议交互流程
date: 2021-02-04 17:00:00 +0800
categories: kafka
tags: kafka
mermaid: true
---

要想理解某个系统是怎么运行的，首先我们可以看看它提供什么样的API。本文从 Kafka 的协议交互流程入手，分析 Producer 和 Consumer 是如何工作的。一方面，可以用来实现自己的 kafkasdk；另一方面也能更好地理解 Kafka 的内部原理。

接下来就从以下3个方面来学习Kafka协议：

- Kafka协议格式，包括编解码方案；
- Producer 工作流程；
- Consumer 工作流程

> 本文基于 Kafka 1.0 版本描述，较新版本(v2.7)肯定有出入，但核心逻辑没有改变

## 1 Kafka协议格式

这里主要参考 Kafka 官方提供的 [KAFKA PROTOCOL GUIDE](http://kafka.apache.org/protocol.html)。如果你要自己实现 kafka Client，那么建议最好把它打印出来放在手边，一个字一个字地看 n 遍。

如果你只是想要了解 Producer 或者 Consumer 的工作流程，那么只需要看看我接下来总结的内容即可。

Kafka协议可以分为 *Request* 和 *Response*。

![Req&Resp](/kafka/req_resp.png)

从某种程度来说，Kafka更多的是提供了 *RPC* 功能：请求只能由 Client 主动发到 Broker；Broker 针对每个请求回复一个响应给 Client。

不同的 Request 使用不同的 `apikey` 来区分；Request 和 Response 通过 `CorrleationId` 来一一对应。

### 编解码方案

从编解码角度来说，每个协议包都是由 4字节的 *size* 开头，后面再跟相应字节的请求包或响应包。

在1.0及以前版本中，Kafka协议中可使用的数据类型仅有3种：

- 固定长度的整形，包括 *int8, int16, int32, int64* 等；
- 可变长度的字符串，包括 *string, bytes* 等；
- 复合类型，包括 *array* 等。

每种类型都有特定的编解码方案，具体可以参考官方文档，这里不再详述。

从2.0版本开始，又增加了很多复杂的类型，比如 *boolean, varint, varlong, uuid, float64, compact_string, compact_bytes* 等等。

### Request & Response

Kafka最新版本中提供的Request已经达到50多种了，但是比较核心的其实也就下面几种：

请求 | 说明
--- | ---
Metadata | 查询集群当前Broker列表，以及指定的topic信息，包括partition数量以及leader/replicas信息
Produce | 发布消息
Fetch | 从指定偏移量开始拉取某个（些）partition的消息
Offset | 查询某个（些）partition的offset信息，可以指定时间戳
Offset Commit | 提交offset，只针对ConsumerGroup
Offset Fetch | 查询某个ConsumerGroup当前提交的offset信息

此外，从0.9版本开始，Kafka提供了消费组的概念，并相应地提供了一组管理协议，包括 *GroupCoordinator/JoinGroup/SyncGroup/LeaveGroup/Heartbeat* 等，具体在后面的Consumer流程中再讲。

#### Request

每个请求都有固定的 *header*，具体格式如下：

```cpp
struct RequestHeader {
    int32_t size; // 请求总长度
    int16_t apikey; // 区分请求类型
    int16_t version; // 区分请求版本
    int32_t correlationId; // 请求上下文，用于对应回包
    std::string clientid; // 请求方标识，仅用来打日志
};
```

在具体发包时，header 在前（这不废话吗！），后面再跟具体的请求包。请求包的大小等于总的 size 减去 header 的大小。

**版本兼容**

注意到头部的 `version` 字段，它是用来保证客户端和服务端版本兼容的。Kafka保证的兼容策略是 *bidirectional compatibility*。即，新版本客户端可以访问旧版本的 broker；新版本 broker 可以接受旧版本的客户端的请求。

客户端在连接上 Broker 并实际开始工作之前，可以先发送 `ApiVersionsRequest` 请求到每个 Broker，以查询 Broker 支持的 版本列表，并从中选取一个它能识别的最高版本作为后续使用版本。

并且这个版本协商是基于连接的：每次连接断开并重连时，都要重新进行版本协商。因为断线可能正是因为Broker升级导致的。

#### Response

每个回包也有固定的 *header*，具体格式如下：

```cpp
struct ResponseHeader
{
    int32_t size;
    int32_t correlationId;
};
```

回包的 header 就很简单了，只有一个 *correlationId*。所以客户端必须要把处理回包时要用到的信息全部在发出请求时保存在请求上下文中，然后通过 *correlationId* 找到上下文。

### C++实现

根据官方文档中的编解码规范，我们就可以自己写一个[C++版本的编解码实现](https://github.com/bookxiao/kafkaprotocpp)了。

kafkaprotocpp的关键类有：

- `Pack` & `Unpack`。负责各种Kafka支持的数据类型的编解码；
- `Request`。负责 Request 的编解码；
- `Response`。负责 Response 的编解码；
- `Marshallable`。所有具体的请求或响应，都需要继承此抽象基类，实现自己的 `marshal`/`unmarshal`方法。

以`MetadataRequest`为例，请求协议定义如下：

```cpp
struct MetadataRequest : public Marshallable
{
    enum { apikey = ApiConstants::METADATA_REQUEST_KEY, apiver = ApiConstants::API_VERSION0};

    std::vector<std::string> vecTopic;

    virtual void marshal(Pack &pk) const
    {
        pk << vecTopic;
    }
    virtual void unmarshal(const Unpack &up)
    {
        up >> vecTopic;
    }

};

struct MetadataResponse : public Marshallable
{
    std::vector<Broker> vecBroker;
    std::vector<TopicMetadata> vecTopicMeta;

    virtual void marshal(Pack &pk) const
    {
        pk << vecBroker << vecTopicMeta;
    }

    virtual void unmarshal(const Unpack &up)
    {
        up >> vecBroker >> vecTopicMeta;
    }
};
```

收发请求如下：

```cpp
std::string topic = "test";

MetadataRequest metareq;
metareq.vecTopic.push_back(topic);
Request req(1, "meta_test", metareq.apikey, metareq.apiver, metareq);

// 发包到指定套接口
int ret = send_request(sockfd, req.data(), req.size());

// 收包
char* buf = read_response(sockfd);
size_t len = Response::peeklen(buf);
Response resp(buf, len);
resp.head();
MetadataResponse meta;
try {
	meta.unmarshall(resp.up);
} catch(PacketError& pe) {
}
// 处理Meta信息
```

感兴趣的可以具体去我的github看它的源码。

## 2 Partition存储模型

在深入了解 Producer 和 Consumer 的交互流程之前，我们先来看下Kafka的存储模型。

在 Kafka 中，*topic* 是消息的逻辑单元：不同的 topic 代表了不同的业务数据，是完全相互独立的。*partition* 则是消息存储的物理单元：每个 topic 可以分成若干个 partition，不同的 partition 可以存储在不同的 Broker 上。

我们先来看下如何在 Kafka 中创建 topic：

```sh
$ bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 2 --partitions 3 --topic test
```

这里我们指定了3个参数：

- `--topic`表示名称，必须唯一；
- `--partitions`表示分区个数，据消息并发吞吐量和客户端处理能力设置；
- `--replication-factor`表示消息备份数，决定了数据的可靠性

接下来可以看到这个topic的具体分区信息：

```sh
$ kafka/bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic test
Topic:test PartitionCount:3	ReplicationFactor:2	Configs:
	Topic: test Partition: 0	Leader: 402	Replicas: 402,401	Isr: 402,401
	Topic: test	Partition: 1	Leader: 403	Replicas: 403,402	Isr: 403,402
	Topic: test	Partition: 2	Leader: 401	Replicas: 401,403	Isr: 401,403
```

当我们在创建topic的时候，kafka会创建指定数量的partition，并将其存储在若干个（由`ReplicationFactor`决定）Broker上。

在这些Broker中，会选出一个作为这个 partition 的 leader，来负责它的生产和消费请求。

> 在Kafka集群中，会有一个Broker被选举为集群的 *Controller* （借助Zookeeper），来负责partition的分配工作，重点是保证集群内各Broker的负载均衡。

例如，这里 `Partition 0` 落在了 *402,401* 这两个Broker上，并且其中排在前面的那个就是它的 Leader。当我们要发布或消费这个 partition 的消息时，必须将 `Producer & Fetch`请求发到这台 Broker。

## 3 Producer 工作流程

接下来看下 Producer 发布消息到 Kafka 时的流程，看看中间都经历了些什么：

![ProducerSeq](/kafka/kafka_producer.png)

这里，我们假定集群有2台Broker，每台Broker各自是一个partition的leader。那么Producer的具体流程可以描述为：

1. Producer向 *BootstrapServer* 发送 `MetadataRequest`，查询集群当前Broker列表，以及partition的leader信息；
2. Producer向 Broker1 请求发布消息到 *partition-0*，收到成功回复；
3. Kafka集群发生 leader 转移，*partition-0* 的 leader 变成了 Broker2；
4. Producer再次向 Broker1 发布消息，收到错误码（*UNKNOWN_TOPIC_OR_PARTITION*）；
5. Producer再次向任意一台 Broker 发送 `MetadataRequest`，查询最新的 leader 信息；
6. Producer重新向 Broker2 请求发布消息到 *partition-0*，收到成功回复。

可以看到，整个发布流程只涉及到两个协议：`MetadataRequest` 和 `ProduceRequest`。

### MetadataRequest

官方格式定义：

```sh
Metadata Request (Version: 0) => [topics]
  topics => name
	name => STRING

Metadata Response (Version: 0) => [brokers] [topics] 
  brokers => node_id host port 
    node_id => INT32
    host => STRING
    port => INT32
  topics => error_code name [partitions] 
    error_code => INT16
    name => STRING
    partitions => error_code partition_index leader_id [replica_nodes] [isr_nodes] 
      error_code => INT16
      partition_index => INT32
      leader_id => INT32
      replica_nodes => INT32
      isr_nodes => INT32
```

`Metadata`主要查询两种信息：

- 集群的Broker列表，包括 *node_id, host, port* 等信息；
- 指定的topic信息，包括 partition 数量，以及每个 partition 的 *leader, replicas, isrs*等。

如果查询时没有指定任何 topic，那么会查询到集群所有的 topic 信息。

Kafka Client 有两种场景下会发送`MetadataRequest`：

- 启动之后定期查询，以便感知到新的Broker信息和topic信息（例如partition扩容了）；
- 当生产或消费时，提示当前Broker已经不是Leader了，需要及时更新信息

正是因为有`Metadata`协议的存在，Kafka Client在运行过程中总是能动态感知到集群所有的Broker信息。因此，我们在启动 Producer 或 Consumer 时，配置的`bootstrap.server`只需要包含一台可用的Broker信息就可以了。

此外，我们通过`MetadataResponse`获取到的 Broker 的 *host* 就是我们在部署 Kafka 时配置的`advertised.listeners`项。需要注意它和`listeners`的区别：

- `listeners`配置的是 Kafka 监听的套接口，例如我们可以配置为`PLAINTEXT://0.0.0.0:9092`来监听本机所有网口；
- `advertised.listeners`是写入到ZooKeeper进而被客户端通过`MetadataRequest`拿到的地址

举个例子，假定我们的Kafka部署在双线或多线机房，为了保证高可用，我们通常是配置为监听所有网口。但是`advertised.listeners`又不能配置为0，所以我们可以给它配置成一个域名：这个域名再绑定Broker的所有网口的具体IP。这样KafkaClient拿到域名后就可以解析到多个IP，并在连接断开时可以尝试使用另外的IP来重连。

### ProduceRequest

官方协议定义：

```sh
v0, v1 (supported in 0.9.0 or later) and v2 (supported in 0.10.0 or later)
ProduceRequest => RequiredAcks Timeout [TopicName [Partition MessageSetSize MessageSet]]
  RequiredAcks => int16
  Timeout => int32
  Partition => int32
  MessageSetSize => int32

v2 (supported in 0.10.0 or later)
ProduceResponse => [TopicName [Partition ErrorCode Offset Timestamp]] ThrottleTime
  TopicName => string
  Partition => int32
  ErrorCode => int16
  Offset => int64
  Timestamp => int64
  ThrottleTime => int32
```

从协议可以看出来，这是一个批量接口：每次请求可以指定发数据给多个 topic 或者多个 partition 。

这里重点看一下 `RequireAcks` 参数，它决定了生产者往Kafka发消息的可靠性。它表示的意思是：Leader 收到请求后需要等待多少个 replicas 的 ack，才能回包给客户端。它的取值为：

- 0，表示Leader不会发送回包给客户端；
- 大于0且小于 ReplicaFactor 的整数，表示将数据同步到指定数量的 Broker 上才给客户端回包；
- -1，表示Leader会将数据同步到所有 ISR 集合中的Broker后才给客户端回包。

所以，`RequireAcks`值越大，表示可靠性越高，但是效率就相应地越低了。不过要更详细地描述可靠性的话，还得理解一个概念 *ISR*。

在Kafka内部，Leader 和 Follower 间的数据同步，依靠每个 Follower 通过 *long-pull* 模式一直不停地从 Leader 拉取数据。那自然地，Leader 会维护每个 Follower 拉取到了哪个 offset，以及与最新offset的差值是多少。当这个差值（lag值）不超过某个既定值时，就认为这个 Follower 是跟 Leader 保持同步的，属于 *In-Sync-Replicas* 集合。

显然，这个 ISR 集合是动态变化的。当某个 Follower 长时间没有过来拉，或者 lag 值比较大时，就会被踢出 ISR；当它恢复之后，又会被重新加入 ISR 集合。只有被 ISR 集合中所有的 Broker 都同步的消息，才被认为是已提交的（*committed*）。只有已提交的消息，才能被消费者看到。

如果当前 leader 挂了，因为已提交的消息肯定在 ISR 集合中的其它 Broker 上都存在，所以只要ISR集合不为空，那么重新选一个作为 Leader 即可。但是如果 ISR 为空呢？这取决于另外一个参数 `unclean.leader.election.enable`：如果设置为 true，那么可以选择一个不在 ISR 集合中的 Replica 作为 leader。但是这可能导致部分已提交的消息丢了，相当于是拿 可靠性 换 可用性。

> 此参数可全局设置，也可针对 topic 设置。

再说回 ProduceRequet。假定我们有个 partition 的 ReplicaFactor 为3，表示会存储在 3 台 Broker（包括Leader）上。如果发布时的 `RequireAcks` 填了 3，那就表示每次发布都要 3 台都同步到数据才算成功。那如果 ISR 集合中没有 3 台会怎样呢？答案就是不可写。

从某种程度来说，我们在创建 topic 时指定的 *ReplicaFactor* 就已经表示了我们对这个 topic 的数据可靠性的要求了。那如果在发布时再去设置ack值，感觉有点冗余了。而且有时候我们在发布的时候，不太好知道这个topic的具体ReplicaFactor值。所以，我们可以将 acks 填为 -1，表示等待当前ISR集合中都同步了就算成功。

在一切正常的情况下，ISR 集合就等于 Replicas 集合；当出问题时，有问题的Broker就会被踢出 ISR 集合。考虑到在不出问题的时候，除Leader之外的Replicas是发挥不出作用的，所以如果没有其它机制保障的话，acks 填 -1，好像可靠性不太“稳定”。

所以 Kafka 提供了另外一个参数`min.insync.replicas`。当 acks 填 -1 时，如果 ISR 集合数量小于此值的话，拒绝写入数据。这样就给可靠性设置了一个底线。

> 此参数可全局设置，也可针对 topic 设置。

## 4 Consumer 工作流程

消费组的工作原理，其实可以分解成3个相互独立的子过程：

- 组关系的维护。包括 *JoinGroup, SyncGroup, LeaveGroup* 以及维持组状态的心跳包 *HeartBeat*；
- offset偏移量的管理。包括 *FetchOffset, FetchCgroupOffset, CommitOffset* 等；
- 拉取消息。包括 *FetchMessage* 等。

这3个过程相互独立，从协议交互角度来说，你可以单独调用每个过程涉及到的协议来实现特定目的。

### 维护消费组关系

**关于Coordinator**

为了保证数据的一致性，每个消费组的状态都由某个固定的 Broker 来管理。这个 Broker 称为该消费组的 *Coordinator*。从负载均衡角度来讲，集群内每个Broker都是差不多数量的消费组的 Coordinator。针对特定消费组来说，它的所有的组管理相关的请求都必须发送给 Coordinator 才能被处理。

因此，Consumer 启动后的第一件事，就是查询它的 Coordinator。方法很简单，发送 `QueryCoordinator` 请求到任意一台 Broker 即可。

**关于消费组**

在一个消费组里，每个 Consumer 都会被 Coordinator 分配一个唯一的 *memberid*。并且，Coordinator会挑选一个 Consumer 作为这个消费组的 *leader*。

所谓消费组，就是多个消费者共同出力来消费某个（些）topic的所有parittion。所有的这些 partition 会被均衡地分配给所有的消费者。也因此，partition 数量一般不会超过消费者的数量。当然，如果只有1个parittion，为了保证高可用，也会起2个消费者，以便当其中1个出问题的时候，另1个能立即接管过来。

那谁来负责在组内分配 partition 呢？你可能会觉得是 Coordinator，但其实不是！真正负责分配的是消费组的 leader Consumer。这也是 Coordinator 的名字的由来：Broker 只是帮忙协调和维护组关系，具体涉及到消费的活（包括分配），都是由 Consumer 自己完成。

**关于消费组状态**

维护组关系，其本质就是在客户端中维护一个消费组的状态机，如下图所示：

![cgroup_fsm](/kafka/cgroup_fsm.png)

在描述状态机转移过程之前，我们先来看一下一个正常的消费组的状态：

每个成员都处于 `CS_UP` 状态，各自消费自己负责的 partition，并且需要每隔固定间隔（不超过配置值 `heartbeat.interval.ms`）发送心跳包给 `Coordinator` 来维持状态，否则就会被踢出消费组。

好了，再来看 Consumer 的状态机转移过程：

- 消费者启动后默认是`CS_DOWN`状态，然后发送`JoinGroup`给 Coordinator 来请求加入组，并转变为`CS_JOINING`状态；
- 当 Coordinator 察觉到成员有变动时，它会在每个现有成员的下一个心跳包回包中告知它们，需要重新发起`JoinGroup`。这样，每个现有成员就从 `CS_UP`状态变为`CS_JOINING`；
- 当 Coordinator 收到所有成员的 Join 请求后，就会从中选出新的 leader，并且通过 JoinResponse 告知所有成员谁是 leader。其中，给 leader 的回包中，会带上所有成员的信息以及它们订阅的topic列表；
- 当消费者收到 JoinResponse 后，就会知道自己是不是 leader。如果是 leader，就需要分配 partition 了，然后把分配结果放在 `SyncGroup` 中发送给 Coordinator。除了 leader 之外，其它成员也都需要发送 `SyncGroup`，只不过不用带分配结果。这样，所有成员都从 `CS_JOINING` 变成了 `CS_SYNCING`；
- Coordinator 收到所有成员的 SyncGroup 请求后，会将 leader 上传的分配结果放到 `SyncResponse` 告知给所有成员。这样所有成员收到回包后，就知道自己该负责消费哪些 partition 了，然后状态从 `CS_SYNCING` 变成 `CS_UP`，就可以准备干活了。
- 此后，每个成员要继续定期发送 `Heartbeat` 以保持在 `CS_UP` 状态。

以上就是整个消费组的状态转移过程了。为了避免一些非法的消费者进程（例如那些卡了很久然后突然又恢复了）干扰消费组状态，Coordinator 会为每个消费组维护一个单调递增的 `generationId`。每次有成员变动时，`generationId`都会加1。当收到`generationId`与当前值不一致的请求时，会拒绝。

**关于消费组偏移量**

当拿到分配结果后，Consumer 就准备开始干活了。

相比于单机版的消费模式，消费组除了能负载均衡之外，还有另外一个好处：Coordinator 可以帮助存储上次消费到的偏移量，以便当某个partition从一个消费者转移到另一个消费者时，可以接着消费，从而保证消息不丢失。

所以，Consumer 在具体拉取消息之前，要首先能够知道该从哪个位置开始拉取。方法也很简单，发送 `OffsetFetchRequest` 到 Coordinator 就可以获取到之前提交的便宜量了。

```sh
v0 and v1 (supported in 0.8.2 or after):
OffsetFetchRequest => ConsumerGroup [TopicName [Partition]]
  ConsumerGroup => string
  TopicName => string
  Partition => int32

v0 and v1 (supported in 0.8.2 or after):
OffsetFetchResponse => [TopicName [Partition Offset Metadata ErrorCode]]
  TopicName => string
  Partition => int32
  Offset => int64
  Metadata => string
  ErrorCode => int16
```

那如果这个消费组之前没人提交过对应 partition 的 offset 呢？

那就需要用另外一个协议了——发送 `OffsetRequest` 到 partition 的 Leader Broker。

```sh
// v0
ListOffsetRequest => ReplicaId [TopicName [Partition Time MaxNumberOfOffsets]]
  ReplicaId => int32
  TopicName => string
  Partition => int32
  Time => int64
  MaxNumberOfOffsets => int32
           
             
// v1 (supported in 0.10.1.0 and later)
ListOffsetRequest => ReplicaId [TopicName [Partition Time]]
  ReplicaId => int32
  TopicName => string
  Partition => int32
  Time => int64
```

这个请求可以查询到指定 partitions 的偏移量信息。其中，参数 `Time` 可以指定要查询哪个时间戳的便宜量，取值可以为：

- `> 0`。表示查询指定时间戳（单位ms）之前的最后一条消息的offset；
- `-1`。表示查询最近的offset（*latest*）；
- `-2`。表示查询最老的offset（*earlist*）。

这也是我们在配置 Consumer 时经常碰到的一个参数：`auto.offset.reset`。只不过比较奇怪的是，明明协议层面可以支持配置一个具体的时间戳，但是所有的Client暴露出来的接口，只能配置成 *earliest* 或 *latest*。

当消费组正常消费时，可以随时把已经消费过的偏移量提交到 Coordinator。方法就是发送 `OffsetCommitRequest` 到 Coordinator。

提交到 Coordinator 的offset信息也是有个有效期的，当超过规定时候没有提交时，Broker 也会把 offset 给删掉的。这样也会重新触发上面提到的没有初始偏移量的逻辑。

这里有个点，就是 Coordinator 不会去校验你提交的offset是否合法。换句话说，它只是提供了一个 key 为 `'groupid/topic/partition'`，value 为 int64 的读写接口。

> 我们可以利用这个特点，来为Kafka跨集群做数据同步。Kafka跨集群同步，方法一般就是在源机房部署一套消费者，然后将消息发布到目的机房。
>
> 那这里就涉及到是采用 同机房消费，跨机房发布 还是 跨机房消费，同机房发布 的选择了。
> 
> - 跨机房消费，意味着消费组的状态不稳定，频繁的网络超时会导致消费组的 rebalance。在消费组 rebalance 时，所有消费都需要暂停。但是跨机房拉取消息本身没有副作用；
> - 跨机房发布，意味着当网络超时时，发布端需要重发，导致目的机房消息重复。(不过新版本Kafka已经支持发布端逻辑去重了)
>
> 不过借助上面提到的特点，我们可以在目的机房部署消费组，但是把消费组的组管理和offset管理放在目的机房Kafka（即在目的机房加入消费组），但是提交的offset其实是源机房partition的offset， 消息还是从源机房拉取。
>
> 这样就综合了两种方案的优点，并且规避了各自的缺点。不过这种方案需要定制化Kafka的SDK。

**关于拉取消息**

有了 partition 列表，有了每个 partition 的初始offset，那么接下来 Consumer 的工作就很简单了，只要通过 *long-pull* 模式不停地去各自 partition 的leader 拉取消息即可。

拉取消息通过发送 `FetchRequest` 给 leader：

```sh
FetchRequest => ReplicaId MaxWaitTime MinBytes [TopicName [Partition FetchOffset MaxBytes]]
  ReplicaId => int32
  MaxWaitTime => int32
  MinBytes => int32
  TopicName => string
  Partition => int32
  FetchOffset => int64
  MaxBytes => int32

v1 (supported in 0.9.0 or later) and v2 (supported in 0.10.0 or later)
FetchResponse => ThrottleTime [TopicName [Partition ErrorCode HighwaterMarkOffset MessageSetSize MessageSet]]
  ThrottleTime => int32
  TopicName => string
  Partition => int32
  ErrorCode => int16
  HighwaterMarkOffset => int64
  MessageSetSize => int32
```

在拉取时，可以指定 `MinBytes` 和 `MaxBytes`，来指定本次拉取最少和最多拉取多少数据，以及最多等待时间 `MaxWaitTime`。在回包中，Kafka除了返回具体的消息之外，还会返回一个参数`HighwaterMarkOffset`，表示该 partition 目前可供消费者消费的最新的offset。通过此值我们可以知道还有多少消息待拉取。

这里需要注意的是，`HighwaterMarkOffset`表示的是消费者能看到的最新offset，不表示发布者最新发布的offset。这个涉及到Kafka内部同步机制，只有被所有 ISR 集合中的Broker同步的消息，才能增加 `HighwaterMarkOffset`。

## 5 总结

除了 Producer 和 Consumer 相关协议之外，Kafka还提供一些管理类的API，包括 `ListGroup`（列出所有消费组）、`DescribeGroups`（查询消费组状态）等等。新版kafka还提供了创建Topic之类的APi。深度利用这些协议可以用来写一些Kafka的监控管理插件。

弄懂Kafka的协议交互流程，除了可以加深对Kafka的理解之外，还有以下的好处：

- 帮助定位生产者和消费者的问题。由于Kafka Broker不会打印任何与客户端相关的异常日志信息，全部都是以错误码的形式告知给客户端。因此了解协议交互流程，就可以更好地从客户端侧的日志了解到具体是哪个环节出问题；
- 实现自己的KafkaSdk。由于作者在工作中需要实现Kafka的跨集群同步数据，而开源的sdk都不太适合，因此只能自己实现了一个C++版sdk；
- 兼容Kafka协议来提供服务。Kafka已经在互联网行业得到大规模应用，如果新开发的MQ系统可以兼容Kafka协议，那么可以掠夺大量的Kafka使用者，实现无缝迁移。例如最近比较火热的 [pulsar](https://pulsar.apache.org/)，就是通过 [Kop](https://github.com/streamnative/kop) 插件来兼容了Kafka协议。

希望本篇文章可以让大家更加理解Kafka的工作模式。
