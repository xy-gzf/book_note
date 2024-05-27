---
description: 生产者
---

# Producer

## 生产者客户端架构

整个生产者客户端由两个线程协调运行，这两个线程分别为主线程和 `Sender` 线程（发送线程）。

在主线程中由 `KafkaProducer` 创建消息，然后通过可能的拦截器、序列化器和分区器的作用之后缓存到消息累加器（`RecordAccumulator`，也称为消息收集器）中。`Sender` 线程负责从 `RecordAccumulator` 中获取消息并将其发送到 Kafka 中。

`RecordAccumulator` 主要用来缓存消息以便 `Sender` 线程可以批量发送，进而减少网络传输的资源消耗以提升性能。`RecordAccumulator` 缓存的大小可以通过生产者客户端参数 `buffer.memory` 配置，默认值为 `33554432B`，即 `32MB`。如果生产者发送消息的速度超过发送到服务器的速度，则会导致生产者空间不足，这个时候 `KafkaProducer` 的 `send（）` 方法调用要么被阻塞，要么抛出异常，这个取决于参数 `max.block.ms` 的配置，此参数的默认值为 60000，即 60 秒。

主线程中发送过来的消息都会被追加到 `RecordAccumulator` 的某个双端队列（Deque）中，在 `RecordAccumulator` 的内部为每个分区都维护了一个双端队列，队列中的内容就是`ProducerBatch`，即 Deque＜ProducerBatch＞。消息写入缓存时，追加到双端队列的尾部；`Sender` 读取消息时，从双端队列的头部读取。

`Sender` 从 `RecordAccumulator` 中获取缓存的消息之后，会进一步将原本＜分区，Deque＜ProducerBatch＞＞ 的保存形式转变成 ＜Node，List＜ProducerBatch＞ 的形式，其中 Node 表示 Kafka 集群的 broker 节点。对于网络连接来说，生产者客户端是与具体的 broker 节点建立的连接，也就是向具体的 broker 节点发送消息，而并不关心消息属于哪一个分区；而对于 KafkaProducer 的应用逻辑而言，我们只关注向哪个分区中发送哪些消息，所以在这里需要做一个应用逻辑层面到网络 I/O 层面的转换。

在转换成 ＜Node，List＜ProducerBatch＞＞ 的形式之后，`Sender` 还会进一步封装成 ＜Node，Request＞ 的形式，这样就可以将 Request 请求发往各个 Node 了，这里的 Request 是指 Kafka 的各种协议请求，对于消息发送而言就是指具体的 ProduceRequest。

请求在从 `Sender` 线程发往 Kafka 之前还会保存到 `InFlightRequests` 中，`InFlightRequests` 保存对象的具体形式为 Map＜NodeId，Deque＜Request＞＞，它的主要作用是缓存了已经发出去但还没有收到响应的请求（NodeId 是一个 String 类型，表示节点的 id 编号）。与此同时，`InFlightRequests` 还提供了许多管理类的方法，并且通过配置参数还可以限制每个连接（也就是客户端与 Node 之间的连接）最多缓存的请求数。这个配置参数为 `max.in.flight.requests.per.connection`，默认值为 `5`，即每个连接最多只能缓存 5 个未响应的请求，超过该数值之后就不能再向这个连接发送更多的请求了，除非有缓存的请求收到了响应（Response）。通过比较 Deque＜Request＞ 的 size 与这个参数的大小来判断对应的 Node 中是否已经堆积了很多未响应的消息，如果真是如此，那么说明这个 Node 节点负载较大或网络连接有问题，再继续向其发送请求会增大请求超时的可能。

## 生产者参数

### acks

`acks` 参数用来指定分区中必须要有多少个副本收到这条消息，之后生产者才会认为这条消息是成功写入的。`acks` 参数目前有三种值，即 `1`、`0` 和 `-1`。接下来我们分别来看看这个参数具体含义。

* `acks=1`：· `acks` 的默认值即为 `1`。生产者发送消息之后，只要分区的 leader 副本成功写入消息，那么它就会收到来自服务端的成功响应。如果消息无法写入 leader 副本，比如在 leader 副本崩溃、重新选举新的 leader 副本的过程中，那么生产者就会收到一个错误的响应，为了避免消息丢失，生产者可以选择重发消息。`acks` 设置为 1，是消息可靠性和吞吐量之间的折中方案。
* `acks=0`：生产者发送消息之后不需要等待任何服务端的响应。如果在消息从发送到写入 Kafka 的过程中出现某些异常，导致 Kafka 并没有收到这条消息，那么生产者也无从得知，消息也就丢失了。在其他配置环境相同的情况下，`acks` 设置为 0 可以达到最大的吞吐量。
* `acks=-1` 或 `acks=all`：生产者在消息发送之后，需要等待 ISR（In-Sync Replicas，同步副本集） 中的所有副本都成功写入消息之后才能够收到来自服务端的成功响应。在其他配置环境相同的情况下，`acks` 设置为 `-1（all）` 可以达到最强的可靠性。但这并不意味着消息就一定可靠，因为 ISR 中可能只有 leader 副本，这样就退化成了 `acks=1` 的情况。

> **在日常工作中，我们会遇到消息中间件（Kafka、Rocketmq 等）会重复发送消息，这个即和 acks 参数有关，acks 参数保证至少有一条消息能够被发送并接收，因此，在日常工作中，我们要注意这种情况，处理好重复消息的问题。**

### max.request.size

`max.request.size` 参数是用来限制生产者客户端能发送的消息的最大值，默认值为 `1048576B`，即 `1MB`。通常情况下，这个默认值就可以满足大多数的应用场景。

### retries 和 retry.backoff.ms

`retries` 参数用来配置生产者重试的次数，默认值为 0，即在发生异常的时候不进行任何重试动作。消息在从生产者发出到成功写入服务器之前可能发生一些临时性的异常，比如网络抖动、 leader 副本的选举等，这种异常往往是可以自行恢复的，生产者可以通过配置 retries 大于 0 的值，以此通过内部重试来恢复而不是一味地将异常抛给生产者的应用程序。如果重试达到设定的次数，那么生产者就会放弃重试并返回异常。

重试还和另一个参数 `retry.backoff.ms` 有关，这个参数的默认值为 `100`，它用来设定两次重试之间的时间间隔，避免无效的频繁重试。

### compression.type

`compression.type` 参数用来指定消息的压缩方式，默认值为“none”，即默认情况下，消息不会被压缩。该参数还可以配置为“gzip”、“snappy”和“lz4”。对消息进行压缩可以极大地减少网络传输量、降低网络 I/O，从而提高整体的性能。消息压缩是一种使用时间换空间的优化方式，如果对时延有一定的要求，则不推荐对消息进行压缩。

### connections.max.idle.ms

`connections.max.idle.ms` 参数用来指定在多久之后关闭限制的连接，默认值是 540000（ms），即 9 分钟。

### linger.ms

`linger.ms` 参数用来指定生产者发送 ProducerBatch 之前等待更多消息（ProducerRecord）加入ProducerBatch 的时间，默认值为 0。生产者客户端会在 ProducerBatch 被填满或等待时间超过 `linger.ms` 值时发送出去。增大这个参数的值会增加消息的延迟，但是同时能提升一定的吞吐量。

### receive.buffer.bytes

`receive.buffer.bytes` 参数用来设置 Socket 接收消息缓冲区（SO\_RECBUF）的大小，默认值为 32768（B），即 32KB。如果设置为 `-1`，则使用操作系统的默认值。如果 Producer 与 Kafka 处于不同的机房，则可以适地调大这个参数值。

### send.buffer.bytes

`send.buffer.bytes` 参数用来设置 Socket 发送消息缓冲区（SO\_SNDBUF）的大小，默认值为 131072（B），即 128KB。与 `receive.buffer.bytes` 参数一样，如果设置为 `-1`，则使用操作系统的默认值。

### request.timeout.ms

`request.timeout.ms` 参数用来配置 Producer 等待请求响应的最长时间，默认值为 30000（ms）。请求超时之后可以选择进行重试。注意这个参数需要比 broker 端参数 `replica.lag.time.max.ms` 的值要大，这样可以减少因客户端重试而引起的消息重复的概率。

