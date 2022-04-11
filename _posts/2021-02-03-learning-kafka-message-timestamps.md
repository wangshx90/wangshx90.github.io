---
title:  Kafka原理学习之消息时间戳
date: 2021-02-02 19:00:00 +0800
categories: kafka
tags: kafka
mermaid: true
---

本文主要讲述kafka对消息时间戳提供的一些支持，以及kafka如何支持根据时间戳精确查找offset。

## 1 前言

得益于kafka良好的设计理念，Producer和Consumer完全独立，互不影响，各司其职即可。但是，对于消费者而言，当它从kafka拿到一条消息时，它可能会想知道，这条消息是何时发布到kafka的呢?

另外，当消费者开始消费时，除了从最新的offset或最久的offset开始之外，是不是可以允许消费者指定回退多长时间来开始消费呢？这在下面的两种场景下会非常有用：

1. 为了保证数据的可靠性，我们通常在异地部署多个相互独立的kafka集群。当消费者从一个集群切到另一个集群时，由于offset不是全局的，所以我们期望切到新集群时，能够回退半小时，以保证消息不丢。
2. 在同个集群内，当我们期望从一个ConsumerGroupId切到另外一个新的ConsumerGroupId后保证消息不丢。为了避免复杂的offset拷贝工作，我们可以让新group直接回退10分钟开始消费即可。

本文主要讲述kafka对消息时间戳提供的一些支持，最终能够解决上面提到的2个问题。

## 2 消息中的时间戳

### 新的消息结构体

在0.10.0版本之后，发布到kafka中的消息结构体中新增了一个时间戳字段。新格式如下：

```cpp
// v1, supported since 0.10.0
class Message {
    int32_t crc;
    int8_t magic_byte;
    int8_t attribute;
    int64_t timestamp;
    Bytes key;
    Bytes value;
};
```

其中，各个字段的意义如下表：

字段名 | 描述
--- | ---
`crc` | crc32完整性校验值
`magic_byte` | Message协议版本号，值为0表示v0版本，值为1表示v1版本
`attribute` | 0-2 bit 消息压缩类型（0 - None, 1 - Gzip, 2 - Snappy）<br> 3 bit 时间戳类型，0表示CreateTime, 1表示LogAppendTime <br> 4-7 预留字段，设为0
`timestamp` | 消息时间戳，其类型取决于`attribute`中相应标志位的值
`key` | 消息的key
`value` | 消息的数据

特别注意到，`Message`依赖`magic_byte`字段实现了新旧协议的兼容性。

### Producer协议

*Message* 是否带有时间戳字段，取决于 *Producer* 在发布时如何填充的。在0.10.0版本中，`ProduerRequest`有3种版本：

```sh
# 不同版本的ProducerRequest没有区别
ProducerRequest => RequiredAcks Timeout [TopicName [Partition MessageSetSize MessageSet]]

# ProducerResponse在不同版本中有区别
# v0
ProducerResponse => [TopicName [Partition ErrorCode Offset]]
# v1
ProducerResponse => [TopicName [Partition ErrorCode Offset]] ThrottleTime
# v2
ProducerResponse => [TopicName [Partition ErrorCode Offset Timestamp]] ThrottleTime
```

> 注：这里采用Kafka官方wiki中对协议的描述语言来表示协议定义，其中'[]'表示array，具体请参考[A Guild To The Kafka Protocol](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol)。

其中，对于v2版本的`ProducerResponse`而言，

- 如果时间戳类型是`CreateTime`，那么返回的`Timestamp`字段就是-1；
- 如果时间戳类型是`LogAppendTime`，那么返回的`Timestamp`字段就是broker将该消息实际写入到log的时间戳。

这里需要强调的是，Message是否带有时间戳，不是取决于`ProducerRequest`的版本号，而是取决于`Message`中的`magic_byte`字段的值。

一般kafka client的做法是先查询broker的版本号，然后选择broker支持的最新的版本号来发送消息。

对于librdkafka来说，发送`Message`时选择的时间戳类型是`CreateTime`。

### Consumer协议

消费者在拉取消息时发送的`FetchRequest`同样也有3种版本：

```sh
# 不同版本的FetchRequest没有区别
FetchRequest => ReplicaId MaxWaitTime MinBytes [TopicName [Partition FetchOffset MaxBytes]]

# FetchResponse在不同版本中有区别
# v0
ffset MessageSetSize MessageSet]]
# v1 & v2
FetchResponse => ThrottleTime [TopicName [Partition ErrorCode HighwaterMarkOffset MessageSetSize MessageSet]]
```

`FetchRequest`中的`MaxWaitTime`字段和`MinBytes`字段可以控制消费者的消费速度：

消费者最多等待`MaxWaitTime`(单位ms)的时间来拉取`MinBytes`字节数的数据。如果在指定的`MaxWaitTime`内没有`MinBytes`的数据可以返回，那么本次请求返回一个空的`MessageSet`。所以消费者在拉取到空的`MessageSet`时，表示已经到了EOF了，循环继续下次请求即可。

*v1* 和 *v2* 版本多了个`ThrottleTime`字段：表示消费者在消费时等了多长时间(单位ms)才拉取到指定字节数的消息。

另外，`FetchResponse`中`HighwaterMarkOffset`字段表示该 *partition* 当前最新的offset值：通过对比这个值和本次拉取到的最大offset，我们可以计算出还有多少消息待消费，可以作为一个评估消费者消费状态的监控指标。

## 3 CreateTime Vs LogAppendTime

上面提到，`Message`结构体中的`Timestamp`有两种类型：`CreateTime`和`LogAppendTime`。具体是何种类型，取决于broker上面的配置：

配置项 | 作用域 | 取值类型| 取值范围 | 默认值
--- | --- | --- | --- | ---
`log.message.timestamp.type` | broker | String | `"CreateTime", "LogAppendTime"` | `"CreateTime"`
`message.timestamp.type` | topic | String | `"CreateTime", "LogAppendTime"` | `"CreateTime"`

如果没有针对topic设置参数的话，那么所有topic使用统一的配置。默认值是`"CreateTime"`，即取自Producer发布时填的值。

- *Producer* 在发布时，因为填的是`CreateTime`，所以`attribute`的相应标志位必须为0；
- *Consumer* 在拉取到消息时，可以根据`attribute`的标志位判断broker上面是否更新了timestamp为`LogAppendTime`。

特别地，对于`CreateTime`类型的topic，broker 在收到消息后，会检查该时间戳跟当前时间戳的差值是否超过`log.message.timestamp.difference.max.ms`配置值。如果超过了该配置值，那么broker会拒绝该消息，并返回错误码给Producer。

配置项 | 作用域 | 取值类型| 取值范围 | 默认值
--- | --- | --- | --- | ---
`log.message.timestamp.difference.max.ms` | broker | long | [0,...] | Long.MaxValue
`message.timestamp.difference.max.ms` | topic | long | [0,...] | Long.MaxValue


## 4 按时间戳查找offset

好了，说明白了消息中的时间戳字段后，我们接下来看看，kafka 是如果按照时间戳来查找offset的。

> 注意，只有0.10.0版本之后的Kafka才支持按照时间戳来查找offset。

我们先上一台 broker 看看数据存储目录(`log.dirs`配置项)中都有些什么。

在`log.dirs`下面，每个partition对应一个子目录，目录名为`$topicname-partition`。在每个 partition 目录下面，有 3 种文件：

- `SegmentBaseOffset.log`：消息存储文件
- `SegmentBaseOffset.index`：消息位置索引，用来根据offset在`.log`中快速找到对应的消息数据
- `SegmentBaseOffset.timeindex`：消息时间戳索引，用来根据时间戳在`.index`中快速找到对应的消息offset

关于`.log`文件和`.index`文件的结构以及作用，可以参考博文[Kafka的Log存储解析](http://blog.csdn.net/jewes/article/details/42970799)。这里我们主要讲`.timeindex`文件的结构以及作用。

### timeindex的结构

`timeindex`文件由一条条的`TimeIndexEntry`组成。每条`TimeIndexEntry`包含两个字段：

```cpp
struct TimeIndexEntry {
	in64_t timestamp;//插入本Entry时本Segment当前最大的消息时间戳
	int32_t offset;//插入本Entry时本Partition的下条Offset值
};
```

对于一条`(Ts, Os)`记录来说，它表示任何在`Ts`之后插入的消息的offset都应该大于`Os`。

### 查找策略

Broker 按照如下步骤，从`timeindex`文件中查找指定的时间戳：

- 先查找最早的segment的`timeindex`文件中的最后（新）一条`TimeIndexEntry`记录；
- 如果该记录的`Ts`大于目标值，那么再对这个`timeindex`文件执行二分查找，直到找到最接近的`TimeIndexEntry`，并返回它的`Os`值；
- 否则，继续检查下一个segment。

查找算法保证了所有晚于目标时间戳的记录都能被返回；早于目标时间戳的记录可能被返回。

我们需要考虑以下几种查找场景：

1. 目标时间戳之前没有消息，但是之后有消息
2. 目标时间戳之后没有消息

- **对于1**，查找算法会返回最早的offset和时间戳，命中的entry应该是最早的segment的第一条entry，所以复杂度应该是O(lg(n))，其中n是第一个segment的entry数量；
- **对于2**，查找算法会**遍历完所有**的segment的timeindex之后**发现无法匹配**，最后返回-1,复杂度是O(mlg(n))，其中m是该partition目录下当前segment的数量。

### 更新timeindex

我们知道，*log index* 并不是针对每条message都会写一条记录到文件，而是每隔固定字节的数据插入一次。这个间隔值是由下面这两个配置值决定的：

配置项 | 作用域 | 取值类型| 取值范围 | 默认值
--- | --- | --- | --- | ---
`log.index.interval.bytes` | broker | int | [0,...] | 4096
`index.interval.bytes` | topic | int | [0,...] | 4096

timeindex 的插入同样受这两个参数控制，因为只有在插入 logindex 的时候才会有可能插入 timeindex。

每次在插入 logindex 的时候，如果当前 timestamp 大于 timeindex 里上一条 entry 的 timestamp 的话，那么就会插入一条新的timeindex entry。

> 注意，如果该topic的Message版本号是0，所以永远不会插入timeindex entry，这类topic的timeindex文件永远是空的。

除了固定间隔时间会插入 logindex 或 timeindex 之外，在 segment rollout 时，会插入一条entry，以保证在index文件里面包含该 segment 最后的offset以及timestamp。

### ListOffsetRequest

kafka提供了`ListOffsetRequest`协议用来查询指定topic+partition的offset，其中v1版本的协议支持按照timestamp精确查找。

```sh
// ListOffsetRequest v0
ListOffsetRequest => ReplicaId [TopicName [Partition Timestamp MaxNumberOfOffsets]]

// ListOffsetRequest v1
ListOffsetRequest => ReplicaId [TopicName [Partition Timestamp]]

// ListOffsetResponse v0
ListOffsetResponse => [TopicName [PartitionOffsets]]
	PartitionOffsets => Partition ErrorCode [Offset]

// ListOffsetResponse v1
ListOffsetResponse => [TopicName [PartitionOffsets]]
	PartitionOffsets => Partition ErrorCode Timestamp Offset
```

虽然v0和v1协议的Request都有一个`Timestamp`字段，但是broker对这两种版本协议的处理方式不一样（只讨论Timestamp > 0）：

- 对于v0，broker根据segment文件(`*.log`）的创建时间来查找，所以精确度其实跟segment的大小有关，最大误差可以达到一整个segment文件；
- 对于v1，broker根据timeindex文件(`*.timeindex`)来实现精确查找，精确度取决于`index.interval.bytes`参数（一般是4kb）。

此外，`Timestamp`还可以设置两个特殊的值去查找：

- `-1`，表示查找最新的offset；
- `-2`，表示查找最久的offset。

回到我们开头提到的问题，对于第一次启动的 Consumer 而言，它只需要在请求 `ListOffsetRequest` 时，请求获取指定时间戳对应的 offset，即可完成回退消费，以保证消息不丢失。
