---
description: kafka
---

# Kafka

### kafka架构

producer 和 customer 可以选定 Kafka 的某些 topic 中投递和消费消息，但 topic 其实只是个逻辑概念，topic 下面分为多个 partition，消息是真正存在 partition 中的：

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FnxQMtXl3G8pUKt37IWUV%2Fuploads%2FLu4mGCMQefsAQGiKW7vR%2Fkafka_topic.jpeg?alt=media&#x26;token=f0621a98-5b99-4c67-926e-c87fc2bed81c" alt=""><figcaption><p>kafka_topic</p></figcaption></figure>

每个 partition 会分配给一个 broker 节点管理：

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FnxQMtXl3G8pUKt37IWUV%2Fuploads%2FLiAEi5rTubhllHWFySTS%2Fkafka_broker.jpeg?alt=media&#x26;token=bdd52967-9e0a-44a7-96ce-393e6e239dd6" alt=""><figcaption><p>kafka_broker</p></figcaption></figure>

所谓 broker 节点，就是一个服务进程。**简单来说，你把一个 broker 节点理解为一台服务器，把 partition 理解为这台服务器上的一个文件就行了**。

发到 topic 的消息实际上是发给了某个 broker 服务器，然后被持久化存储到一个文件里，我们一般称这个文件是 log file。

那么为什么要给一个 topic 分配多个 partition 呢？

很显然，如果一个 topic 只有一个 partition，那么也就只能有一台 broker 服务器处理这个 topic 上的消息，如果划分成很多 partition，就可以把这个 topic 上的消息分配到多台 broker 的 partition 上，每个 broker 处理消息并将消息持久化存储到 log file 中，从而提高单 topic 的数据处理能力。

但问题是怎么保证高可用？如果某个 broker 节点挂了，对应的 partition 上的数据不就就无法访问了吗？

一般都是通过「数据冗余」和「故障自动恢复」来保证高可用，Kafka 会对每个 partition 维护若干冗余副本：

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FnxQMtXl3G8pUKt37IWUV%2Fuploads%2Fbo0zvamsv0ySHXxQw2Y9%2Fkafka_follower.jpeg?alt=media&#x26;token=6d8c7bd1-a33f-4228-8377-c8f2e35acae5" alt=""><figcaption><p>kafka_follower</p></figcaption></figure>

若干个 partition 副本中，有一个 leader 副本（图中红色的），其余都是 follower 副本（也可称为 replica，图中橙色的）。

leader 副本负责向生产者和消费者提供订阅发布服务，负责持久化存储消息，同时还要把最新的消息同步给所有 follower，让 follower 小弟们和自己存储的数据尽可能相同。

这样的话，如果 leader 副本挂了，就能从 follower 中选取一个副本作为新的 leader，继续对外提供服务。

### kafka架构缺陷

#### **1、Kafka 把 broker 和 partition 的数据存储牢牢绑定在一起，会产生很多问题。**

首先一个问题就是，Kafka 的很多操作都涉及 partition 数据的全量复制。

比方说典型的扩容场景，假设有`broker1, broker2`两个节点，它们分别负责若干个 partition，现在我想新加入一个`broker3`节点分摊`broker1`的部分负载，那你得让`broker1`拿一些 partition 给到`broker3`对吧？

那不好意思，得复制 partition 的全量数据，什么时候复制完，`broker3`才能上线提供服务，这会消耗大量 IO 以及网络资源。

再比方说，我想给某个 partition 新增一个 follower 副本，那么这个新增的 follower 副本必须要跟 leader 副本同步全量数据。毕竟 follower 存在的目的就是随时替代 leader，所以复制 leader 的全量数据是必须的。

除此之外，因为 broker 要负责存储，所以整个集群的容量可能局限于存储能力最差的那个 broker 节点。而且如果某些 partition 中的数据特别多（数据倾斜），那么对应 broker 的磁盘可能很快被写满，这又要涉及到 partition 的迁移，数据复制在所难免。

虽然 Kafka 提供了现成的脚本来做这些事情，但实际需要考虑的问题比较多，操作也比较复杂，数据迁移也很耗时，远远做不到集群的「平滑扩容」。

**2、Kafka 底层依赖操作系统的 Page Cache，会产生很多问题**。

之前说了，只有数据被写到磁盘里才能保证万无一失，否则的话都不能保证数据不会丢。所以首先一个问题就是 Kafka 消息持久化并不可靠，可能丢消息。

我们知道 Linux 文件系统会利用 Page Cache 机制优化性能。Page Cache 说白了就是读写缓存，Linux 告诉你写入成功，但实际上数据并没有真的写进磁盘里，而是写到了 Page Cache 缓存里，可能要过一会儿才会被真正写入磁盘。

那么这里面就有一个时间差，当数据还在 Page Cache 缓存没有落盘的时候机器突然断电，缓存中的数据就会永远丢失。

而 Kafka 底层完全依赖 Page Cache，并没有要求 Linux 强制刷盘，所以突然断电的情况是有可能导致数据丢失的。虽然对于大部分场景来说可以容忍偶尔丢点数据，但对于金融支付这类服务场景，是绝对不能接受丢数据的。

另外，虽然我看到很多博客都把 Kafka 依赖 page cache 这个特性看做是 Kafka 的优点，理由是可以提升性能，但实际上 Page Cache 也是有可能出现性能问题的。

我们来分析下消费者消费数据的情况，主要有两种可能：一种叫**追尾读**（Tailing Reads），一种叫**追赶读**（Catch-up Reads）。

所谓**追尾读**，顾名思义，就是消费者的消费速度比较快，生产者刚生产一条消息，消费者立刻就把它消费了。我们可以想象一下这种情况 broker 底层是如何处理的：

生产者写入消息，broker 把消息写入 Page Cache 写缓存，然后消费者马上就来读消息，那么 broker 就可以快速地从 Page Cache 里面读取这条消息发给消费者，这挺好，没毛病。

所谓**追赶读**的场景，就是消费者的消费速度比较慢，生产者已经生产了很多新消息了，但消费者还在读取比较旧的数据。

这种情况下，Page Cache 缓存里没有消费者想读的老数据，那么 broker 就不得不从磁盘中读取数据并存储在 Page Cache 读缓存。

注意此时读写都依赖 Page Cache，所以读写操作可能会互相影响，对一个 partition 的大量读可能影响到写入性能，大量写也会影响读取性能，而且读写缓存会互相争用内存资源，可能造成 IO 性能抖动。

再进一步分析，因为每个 partition 都可以理解为 broker 节点上的一个文件，那么如果 partition 的数量特别多，一个 broker 就需要同时对很多文件进行大量读写操作，这性能可就……
