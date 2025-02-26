---
description: 深入服务端
---

# Explore Server

## 协议设计

在 Kafka 2.0.0 中，一共包含了 43 种协议类型，每种协议类型都有对应的请求（Request）和响应（Response），它们都遵守特定的协议模式。每种类型的 Request 都包含相同结构的协议请求头（RequestHeader）和不同结构的协议请求体（RequestBody）

### 协议结构

#### 请求结构

**协议请求头中包含 4 个域（Field）：`api_key`、`api_version`、`correlation_id`和`client_id`**，其含义如下：

* `api_key`：API 标识，比如 PRODUCE、FETCH 等分别表示发送消息和拉取消息的请求；
* `api_version`：API 版本号；
* `correlation_id`：由客户端指定的一个数字来唯一标识这次请求的 ID，服务端在处理完请求后也会把同样的 correlation\_id 写到 Response 中，这样客户端就能把某个请求和响应对应起来了；
* `client_id`：客户端 ID；

协议请求体中各个域含义如下：

* `transction_id`：事务 ID，从 Kafka 0.11.0 开始支持事务，如果不使用事务的功能，那么该域的值为 NULL；
* `acks`：对应客户端参数 acks；
* `timeout`：请求超时时间，对应客户端参数 request.timeout.ms，默认值为 30000，即 30 秒；
* `topic data`：array 类型，代表 ProduceRequest 中所要发送的数据集合。以主体名称分类，主题中再以分区分类；
  * `topic`：主体名称
  * `data`：array 类型，与主题对应对的数据
    * `patition`：分区编号
    * `record_set`：与分区对应的数据

#### 响应结构

**协议响应头中只有一个 `correlation_id`**

除了响应头中的 correlation\_id，其余 ProduceResponse 各个域的字段如下：

* `throttle_time_ms`：如果超过了配额（quota）限制则需要延迟该请求的处理时间。如果没有配置配额，那么该字段的值为 0；
* `response`：该字段类型为 array，代表 ProduceResponse 中要返回的数据集合，同样按照主题分区的粒度进行划分；
  * `topic`：主体名称
  * `patition_response`：array 类型，主题中所有分区的响应
    * `patition`：分区编号
    * `error_code`：错误码，用来标识错误类型，目前版本的错误码有 74 种；
    * `base_offset`：消息集的起始偏移量
    * `log_append_time`：消息写入 broker 端的时间
    * `log_start_offset`：所在分区的起始偏移量；

消息追加是针对单个分区而言的，那么响应也是针对分区粒度来进行划分的，这样 ProduceRequest 和 ProduceResponse 做到了一一对应。

## 时间轮

Kafka 中存在大量的延时操作，比如延时生产、延时拉取和延时删除等。其实现方式是基于时间轮实现的一个用于延时功能的定时器（SystemTimer）。相比于 JDK 中 Timer 和 DelayQueue 延时功能组件，其时间复杂度更高，为 O(1)，更能满足 Kafka 的高性能要求，而 Timer 和 DelayQueue 的插入和删除操作的平均时间复杂度为 O（nlogn）

时间轮在很多地方都有应用，如zk，netty等

### 时间轮

**Kafka 中的时间轮（TimingWheel）是一个 存储定时任务的环形队列，底层采用数组实现，数组中的每个元素可以存放一个定时任务列表（TimerTaskList）。TimerTaskList 是一个环形的双向链表，链表中的每一项表示的都是定时任务项（TimerTaskEntry），其中封装了真正的定时任务（TimerTask）。**

时间轮由多个 `时间格` 组成，其结构如下图所示。每个时间格代表当前时间轮的 `基本时间跨度（tickMs）`。时间轮的时间格个数是固定的，可用 wheelSize 来表示，那么整个时间轮的总体时间跨度（interval）可以通过公式 `tickMs × wheelSize` 计算得出。时间轮还有一个 `表盘指针（currentTime）`，用来表示时间轮当前所处的时间，currentTime 是 tickMs 的整数倍。currentTime 可以将整个时间轮划分为 `到期部分` 和 `未到期部分`，**currentTime 当前指向的时间格也属于到期部分，表示刚好到期，需要处理此时间格所对应的 TimerTaskList 中的所有任务**。

### 时间轮使用

#### 创建第一层时间轮

若时间轮的 tickMs 为 1ms 且 wheelSize 等于 20，那么可以计算得出总体时间跨度 interval 为 20ms。初始情况下表盘指针 currentTime 指向时间格 0。

#### 添加任务

此时有一个定时为 2ms 的任务插进来会存放到时间格为 2 的 TimerTaskList 中。随着时间的不断推移，指针 currentTime 不断向前推进，过了 2ms 之后，当到达时间格 2 时，就需要将时间格 2 对应的 TimeTaskList 中的任务进行相应的到期操作。

此时若又有一个定时为 8ms 的任务插进来，则会存放到时间格 10 中，currentTime 再过 8ms 后会指向时间格 10。

如果同时有一个定时为 19ms 的任务插进来怎么办？新来的 TimerTaskEntry 会复用原来的 TimerTaskList，所以它会插入原本已经到期的时间格 1。

总之，整个时间轮的总体跨度是不变的，随着指针 currentTime 的不断推进，当前时间轮所能处理的时间段也在不断后移，总体时间范围在 currentTime 和 currentTime+interval 之间。

#### 添加超过wheelSize的任务

那我们再来看一看，如果此时有一个定时为 350ms 的任务该如何处理？直接扩充 wheelSize 的大小肯定是不合理的，扩充是没有底线，就算将所有的定时任务的到期时间都设定一个上限，比如 100 万毫秒，那么这个 wheelSize 为 100 万毫秒的时间轮不仅占用很大的内存空间，而且也会拉低效率。Kafka 为此引入了 `层级时间轮` 的概念，当任务的到期时间超过了当前时间轮所表示的时间范围时，就会尝试添加到上层时间轮中。

即第一层的时间轮 tickMs=1ms、wheelSize=20、interval=20ms。第二层的时间轮的 tickMs 为第一层时间轮的 interval，即 20ms。**每一层时间轮的 wheelSize 是固定的，都是 20**，那么第二层的时间轮的总体时间跨度 interval 为 400ms。以此类推，这个 400ms 也是第三层的 tickMs 的大小，第三层的时间轮的总体时间跨度为 8000ms

对于之前所说的 350ms 的定时任务，显然第一层时间轮不能满足条件，所以就升级到第二层时间轮中，最终被插入第二层时间轮中时间格 17 所对应的 TimerTaskList。

如果此时又有一个定时为 450ms 的任务，那么显然第二层时间轮也无法满足条件，所以又升级到第三层时间轮中，最终被插入第三层时间轮中时间格 1 的 TimerTaskList。

### 时间轮降级

注意到在到期时间为 \[400ms，800ms）区间内的多个任务（比如 446ms、455ms 和 473ms 的定时任务）都会被放入第三层时间轮的时间格 1，时间格 1 对应的 TimerTaskList 的超时时间为 400ms。随着时间的流逝，当此 TimerTaskList 到期之时，原本定时为 450ms 的任务还剩下 50ms 的时间，还不能执行这个任务的到期操作。

这里就有一个 `时间轮降级` 的操作，会将这个剩余时间为 50ms 的定时任务重新提交到层级时间轮中，此时第一层时间轮的总体时间跨度不够，而第二层足够，所以该任务被放到第二层时间轮到期时间为 \[40ms，60ms）的时间格中。再经历 40ms 之后，此时这个任务又被“察觉”，不过还剩余 10ms，还是不能立即执行到期操作。所以还要再有一次时间轮的降级，此任务被添加到第一层时间轮到期时间为 \[10ms，11ms）的时间格中，之后再经历 10ms 后，此任务真正到期，最终执行相应的到期操作。

### 时间轮实现细节

Kafka 在实现时间轮时还有一些小细节，这里我们一起看下：

1. **起始时间初始化**： TimingWheel 在创建的时候以当前系统时间为第一层时间轮的起始时间（startMs），时间设置是调用 Time.SYSTEM.hiResClockMs 方法，该方法可以精确的获取毫秒级时间，不依赖于具体操作系统。
2. **哨兵节点**：TimingWheel 中的每个双向环形链表 TimerTaskList 都会有一个哨兵节点（sentinel），引入哨兵节点可以简化边界条件。哨兵节点也称为哑元节点（dummy node），它是一个附加的链表节点，该节点作为第一个节点，它的值域中并不存储任何东西，只是为了操作的方便而引入的。如果一个链表有哨兵节点，那么线性表的第一个元素应该是链表的第二个节点。
3. **层级起始时间**：除了第一层时间轮，其余高层时间轮的起始时间（startMs）都设置为创建此层时间轮时前面第一轮的 currentTime。每一层的 currentTime 都必须是 tickMs 的整数倍，如果不满足则会将 currentTime 修剪为 tickMs 的整数倍，以此与时间轮中的时间格的到期时间范围对应起来。修剪方法为：`currentTime=startMs-（startMs%tickMs）`。currentTime 会随着时间推移而推进，但不会改变为 tickMs 的整数倍的既定事实。若某一时刻的时间为 timeMs，那么此时时间轮的 currentTime=timeMs-（timeMs%tickMs），时间每推进一次，每个层级的时间轮的 currentTime 都会依据此公式执行推进。
4. **时间轮引用**：Kafka 中的定时器只需持有 TimingWheel 的第一层时间轮的引用，并不会直接持有其他高层的时间轮，但每一层时间轮都会有一个引用（overflowWheel）指向更高一层的应用，以此层级调用可以实现定时器间接持有各个层级时间轮的引用。

## 延时操作

`acks` 参数用来指定分区中必须要有多少个副本收到这条消息，之后生产者才会认为这条消息是成功写入的。`acks` 参数目前有三种值，即 `1`、`0` 和 `-1`

`acks=-1` 或 `acks=all`：那么意味着需要等待ISR中所有副本都确认收到消息才能正确收到相应结构，或者捕捉超时异常。

那么等待消息写入副本和返回相应的响应结果给客户端这个动作是由谁来执行的呢？答案是 `在将消息写入 leader 副本的本地日志文件之后，Kafka会创建一个延时的生产操作（DelayedProduce），用来处理消息正常写入所有副本或超时的情况，以返回相应的响应结果给客户端。`

在 Kafka 中有多种延时操作，比如延时生产，延时拉取（DelayedFetch）、延时数据删除（DelayedDeleteRecords）等。延时操作需要延时返回响应的结果，首先它必须有一个超时时间（delayMs），如果在这个超时时间内没有完成既定的任务，那么就需要强制完成以返回响应结果给客户端。其次，延时操作不同于定时操作，定时操作是指在特定时间之后执行的操作，而延时操作可以在所设定的超时时间之前完成，所以 `延时操作能够支持外部事件的触发`。

延时操作创建之后会被加入延时操作管理器（DelayedOperationPurgatory）来做专门的处理。延时操作有可能会超时，每个延时操作管理器都会配备一个定时器（SystemTimer）来做超时管理，定时器的底层就是采用时间轮（TimingWheel）实现的。

### 延时生产

对于延时生产操作而言，它的外部事件是 `写入消息的某个分区的 HW（高水位）发生增长`。也就是说，随着 follower 副本不断地与 leader 副本进行消息同步，进而促使 HW 进一步增长，HW 每增长一次都会检测是否能够完成此次延时生产操作，如果可以就执行以此返回响应结果给客户端；如果在超时时间内始终无法完成，则强制执行。

### 延时拉取

延时拉取操作同样是由 `超时触发` 或 `外部事件触发` 而被执行的。

超时触发很好理解，就是等到超时时间之后触发第二次读取日志文件的操作。

外部事件触发就稍复杂了一些，因为拉取请求不单单由 follower 副本发起，也可以由消费者客户端发起，两种情况所对应的外部事件也是不同的。如果是 follower 副本的延时拉取，它的外部事件就是消息追加到了 leader 副本的本地日志文件中；如果是消费者客户端的延时拉取，它的外部事件可以简单地理解为 HW 的增长。

## 控制器

在 Kafka 集群中会有一个或多个 broker，其中有一个 broker 会被选举为控制器（Kafka Controller），它负责管理整个集群中所有分区和副本的状态。比如当某个分区的 leader 副本出现故障时，由控制器负责为该分区选举新的 leader 副本。当检测到某个分区的 ISR 集合发生变化时，由控制器负责通知所有 broker 更新其元数据信息。当使用 kafka-topics.sh 脚本为某个 topic 增加分区数量时，同样还是由控制器负责分区的重新分配。

### 控制器选举

Kafka 中的控制器选举工作依赖于 ZooKeeper，成功竞选为控制器的 broker 会在 ZooKeeper 中创建 /controller 这个临时（EPHEMERAL）节点，此临时节点的内容参考如下：

```json
{"version":1, "brokerid":0, "timestamp": "123123213123"}
```

其中 version 在目前版本中固定为 1，brokerid 表示成为控制器的 broker 的 id 编号，timestamp 表示竞选成为控制器时的时间戳。

**`注意：在任意时刻，集群中有且仅有一个控制器`。**

每个 broker 启动的时候会去尝试读取 /controller 节点的 `brokerid` 的值，如果读取到 brokerid 的值不为 -1，则表示已经有其他 broker 节点成功竞选为控制器，所以当前 broker 就会放弃竞选；如果 ZooKeeper 中不存在 /controller 节点，或者这个节点中的数据异常，那么就会尝试去创建 /controller 节点。当前 broker 去创建节点的时候，也有可能其他 broker 同时去尝试创建这个节点，只有创建成功的那个 broker 才会成为控制器，而创建失败的 broker 竞选失败。每个 broker 都会在内存中保存当前控制器的 brokerid 值，这个值可以标识为 activeControllerId。

ZooKeeper 中还有一个与控制器有关的 /controller\_epoch 节点，这个节点是持久（PERSISTENT）节点，节点中存放的是一个整型的 controller\_epoch 值。controller\_epoch 用于记录控制器发生变更的次数，即记录当前的控制器是第几代控制器，我们也可以称之为“控制器的纪元”。

controller\_epoch的初始值为1，即集群中第一个控制器的纪元为 1，当控制器发生变更时，每选出一个新的控制器就将该字段值加 1。每个和控制器交互的请求都会携带 controller\_epoch 这个字段：

* 如果请求的 controller\_epoch 值`小于`内存中的 controller\_epoch 值，则认为这个请求 `是向已经过期的控制器所发送的请求，那么这个请求会被认定为无效的请求`；
* 如果请求的 controller\_epoch 值 `大于` 内存中的 controller\_epoch 值，那么 `说明已经有新的控制器当选了`。

**`由此可见，Kafka 通过 controller_epoch 来保证控制器的唯一性，进而保证相关操作的一致性。`**

### 控制器作用

下面我们再来看一看，具备控制器身份的 broker 比其他普通的 broker 多了哪些作用。

1. `监听分区相关的变化`，包括：
   * 为 ZooKeeper 中的 /admin/reassign\_partitions 节点注册 PartitionReassignmentHandler，用来处理 `分区重分配` 的动作。
   * 为 ZooKeeper 中的 /isr\_change\_notification 节点注册 IsrChangeNotificetionHandler，用来处理 `ISR 集合变更` 的动作。
   * 为 ZooKeeper中的 /admin/preferred-replica-election 节点添加 PreferredReplicaElectionHandler，用来处理 `优先副本选举` 动作。
2. `监听主题相关的变化`，包括：
   * 为 ZooKeeper 中的 /brokers/topics 节点添加 TopicChangeHandler，用来处理 `主题增减` 的变化；
   * 为 ZooKeeper 中的 /admin/delete\_topics 节点添加 TopicDeletionHandler，用来处理 `删除主题` 的动作。
3. `监听 broker 相关的变化`：
   * 为 ZooKeeper 中的 /brokers/ids 节点添加 BrokerChangeHandler，用来处理 `broker 增减` 的变化。
4. `从 ZooKeeper 中读取获取当前所有与主题、分区及 broker 有关的信息并进行相应的管理`：
   * 对所有主题对应的 ZooKeeper 中的 /brokers/topics/＜topic＞ 节点添加 PartitionModificationsHandler，用来 `监听主题中的分区分配变化`。
5. `启动并管理分区状态机和副本状态机`
6. `更新集群的元数据信息`
7. 如果参数 auto.leader.rebalance.enable 设置为 true，则还会开启一个名为“auto-leader-rebalance-task”的定时任务来负责 `维护分区的优先副本的均衡`。

### 线程安全

控制器在选举成功之后会读取 ZooKeeper 中各个节点的数据来 **初始化上下文信息**，并且需要管理这些上下文信息。比如为某个主题增加了若干分区，控制器在负责创建这些分区的同时要更新上下文信息，并且需要将这些变更信息同步到其他普通的 broker 节点中。

不管是监听器触发的事件，还是定时任务触发的事件，或者是其他事件都会读取或更新控制器中的上下文信息，那么这样就会涉及多线程间的同步。如果单纯使用锁机制来实现，那么整体的性能会大打折扣。

针对这一现象，Kafka 的控制器使用 `单线程基于事件队列的模型，将每个事件都做一层封装，然后按照事件发生的先后顺序暂存到 LinkedBlockingQueue 中，最后使用一个专用的线程按照FIFO（First Input First Output，先入先出）的原则顺序处理各个事件，这样不需要锁机制就可以在多线程间维护线程安全`。
