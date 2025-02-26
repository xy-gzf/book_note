---
description: 日志存储
---

# Log

> kafka的消息是以主题为基本单位归类，每个主题在逻辑上相互独立，每个主题分为一个或多个分区，分区数量可以创建时指定也可以之后修改。

## 文件目录布局

不考虑多副本的情况，一个分区对应一个**日志（Log）**。为了防止Log过大，kafka引入了**日志分段（LogSegment）**&#x7684;概念，将Log切分为多个LogSegment。而Log和LogSegment也不是纯粹的物理意义。Log在物理上只以文件夹形式存储，每个LogSegment对应为磁盘上的一个日志文件和两个索引文件，以及可能的其他文件（如以".txnindex"为后缀的事务索引文件及 .swap .cleaned 等临时文件）。

**日志关系如下：**

* Topic 主题
  * Partition 分区
    * Replica（副本）-> Log 日志
      * LogSegment 日志分段（.log .index .timeindex 其他文件）

**文件内容：**&#x5411;Log中追加消息是顺序写入，只有最后一个LogSegment才能执行写入操作，此LogSegment可以称为activeSegment。每个LogSegment都有一个基准偏移量baseOffset来表示当前LogSegment的第一条消息的offset。**.log 日志文件 .index 偏移量索引文件 .timeindex 时间戳索引文件**

**偏移量**是一个64位的长整型数，日志文件及两个索引文件都是基于baseOffset命名，名称固定为20位数字，没有达到则用0填充。

**注意：**

* 消费者的位移提交是保存在内部主题\_\_consumer\_offsets中，初始该主题不存在，当第一次有消费者消费消息时自动创建。

## 日志格式

### v0版本

<figure><img src="https://1301551370.vod-qcloud.com/dc9dab3cvodcq1301551370/2e6689671253642696686037134/wN3CMhHepaAA.png" alt=""><figcaption><p>v0版本消息格式</p></figcaption></figure>

每个Record（v0和v1版）必定对应一个 offset和message size。每条消息都一个offset用来标志它在partition中的偏移量，这个offset是逻辑值，而非实际物理偏移值， message size表示消息的大小，这两者的一起被称之为日志头部（LOG\_OVERHEAD），固定为12B。LOG\_OVERHEAD和RECORD一起用来描述一条消息。&#x20;

与消息对应的还有消息集的概念，消息集中包含一条或者多条消息，消息集不仅是存储于磁盘以及在网络上传输（Produce & Fetch）的基本形式， 而且是kafka中压缩的基本单元，详细结构参考上右图。

<details>

<summary>字段具体解释</summary>

从crc32开始算起，各个字段的解释如下：

* crc32（4B）：crc32校验值。校验范围为magic至value之间。
* magic（1B）：消息格式版本号，此版本的magic值为0。
* attributes（1B）：消息的属性。总共占1个字节，低3位表示压缩类型：0表示NONE、1表示GZIP、2表示SNAPPY、3表示LZ4（LZ4自Kafka 0.9.x引入），其余位保留。
* key length（4B）：表示消息的key的长度。如果为-1，则表示没有设置key，即key=null。
* key：可选，如果没有key则无此字段。
* value length（4B）：实际消息体的长度。如果为-1，则表示消息为空。
* value：消息体。可以为空，比如tomnstone消息。

v0版本中一个消息的最小长度（RECORD\_OVERHEAD\_V0）为crc32 + magic + attributes + key length + value length = 4B + 1B + 1B + 4B + 4B =14B， 也就是说v0版本中一条消息的最小长度为14B，如果小于这个值，那么这就是一条破损的消息而不被接受。

</details>

### v1版本

<figure><img src="https://1301551370.vod-qcloud.com/3b912203vodsh1301551370/7ba928311253642696687077139/FrdaPihM5KgA.png" alt=""><figcaption><p>v1版本消息结构</p></figcaption></figure>

kafka从0.10.0版本开始到0.11.0版本之前所使用的消息格式版本为v1，其比v0版本就多了一个timestamp字段，表示消息的时间戳

1版本的attributes第4个bit被利用了起来：0表示timestamp类型为CreateTime（默认）， 而1表示tImestamp类型为LogAppendTime，其他位保留。v1版本的最小消息（RECORD\_OVERHEAD\_V1）大小要比v0版本的要大8个字节，即22B。

### 消息压缩

kafka实现的压缩方式是将多条消息一起进行压缩， 这样可以保证较好的压缩效果。一般情况下，**生产者发送的压缩数据在kafka broker中也是保持压缩状态进行存储，消费者从服务端获取也是压缩的消息， 消费者在处理消息之前才会解压消息。**

当消息压缩时是将整个消息集进行压缩而作为内层消息（inner message），内层消息整体作为外层（wrapper message）的value，其结构图如下所示：

<figure><img src="https://1301551370.vod-qcloud.com/dc9dab3cvodcq1301551370/5f3ac0e71253642696690801842/VaPmZR3T8FUA.png" alt=""><figcaption><p>消息压缩</p></figcaption></figure>

压缩后的外层消息（wrapper message）中的key为null，所以图右部分没有画出key这一部分。当生产者创建压缩消息的时候， 对内部压缩消息设置的offset是从0开始为每个内部消息分配offset

<figure><img src="https://1301551370.vod-qcloud.com/dc9dab3cvodcq1301551370/513d54191253642696690204111/RfmAaaSub4sA.png" alt=""><figcaption><p>外层消息和内层消息</p></figcaption></figure>

每个从生产者发出的消息集中的消息offset都是从0开始的，当然这个offset不能直接存储在日志文件中，对offset进行转换时在服务端进行的， 客户端不需要做这个工作。外层消息保存了内层消息中最后一条消息的绝对位移（absolute offset），绝对位移是指相对于整个partition而言的。 参考上图，对于未压缩的情形，图右内层消息最后一条的offset理应是1030，但是被压缩之后就变成了5，而这个1030被赋予给了外层的offset。 当消费者消费这个消息集的时候，首先解压缩整个消息集，然后找到内层消息中最后一条消息的inner offset， 然后根据如下公式找到内层消息中最后一条消息前面的消息的absolute offset（RO表示Relative Offset，IO表示Inner Offset，而AO表示Absolute Offset）：

```
RO = IO_of_a_message - IO_of_the_last_message 
AO = AO_Of_Last_Inner_Message + RO
```

<details>

<summary>RO</summary>

这里RO是前面的消息相对于最后一条消息的IO而言的，所以其值小于等于0，0表示最后一条消息自身。

v1版本比v0版的消息多了个timestamp的字段。对于压缩的情形，外层消息的timestamp设置为：

* 如果timestamp类型是CreateTime，那么设置的是内层消息中最大的时间戳（the max timestampof inner messages if CreateTime is used）
* 如果timestamp类型是LogAppendTime，那么设置的是kafka服务器当前的时间戳

内层消息的timestamp设置为：

* 如果外层消息的timestamp类型是CreateTime，那么设置的是生产者创建消息时的时间戳。
* 如果外层消息的timestamp类型是LogAppendTime，那么所有的内层消息的时间戳都将被忽略。

</details>

### v2版本

kafka从0.11.0版本开始所使用的消息格式版本为v2，参考了Protocol Buffer而引入了变长整型（Varints）和ZigZag编码。 Varints是使用一个或多个字节来序列化整数的一种方法，数值越小，其所占用的字节数就越少。ZigZag编码以一种锯齿形（zig-zags）的方式来回穿梭于正负整数之间， 以使得带符号整数映射为无符号整数，这样可以使得绝对值较小的负数仍然享有较小的Varints编码值，比如-1编码为1,1编码为2，-2编码为3。

kafka v0和v1版本的消息格式，如果消息本身没有key，那么key length字段为-1，int类型的需要4个字节来保存，而如果采用Varints来编码则只需要一个字节。 根据Varints的规则可以推导出0-63之间的数字占1个字节，64-8191之间的数字占2个字节，8192-1048575之间的数字占3个字节。 而kafka broker的配置message.max.bytes的默认大小为1000012（Varints编码占3个字节），如果消息格式中与长度有关的字段采用Varints的编码的话， 绝大多数情况下都会节省空间，而v2版本的消息格式也正是这样做的。不过需要注意的是Varints并非一直会省空间，一个int32最长会占用5个字节（大于默认的4字节）， 一个int64最长会占用10字节（大于默认的8字节）。

v2版本中消息集为Record Batch，而不是先前的Message Set了，其内部也包含了一条或者多条消息，消息的格式参见下图中部和右部。在消息压缩的情形下， Record Batch Header部分（参见下图左部，从first offset到records count字段）是不被压缩的，而被压缩的是records字段中的所有内容。

<figure><img src="https://1301551370.vod-qcloud.com/3b912203vodsh1301551370/4d0143881253642696690067317/wYaLQYO8z2cA.png" alt=""><figcaption><p>v2版本消息结构</p></figcaption></figure>

<details>

<summary>v2版本对于消息集（RecordBatch）做了彻底的修改，参考上图左部，如下字段：</summary>

* first offset：表示当前RecordBatch的起始位移。
* length：计算partition leader epoch到headers之间的长度。
* partition leader epoch：用来确保数据可靠性，详细可以参考KIP-101
* magic：消息格式的版本号，对于v2版本而言，magic等于2。
* attributes：消息属性，注意这里占用了两个字节。低3位表示压缩格式，可以参考v0和v1；第4位表示时间戳类型；第5位表示此RecordBatch是否处于事务中，0表示非事务，1表示事务。第6位表示是否是Control消息，0表示非Control消息，而1表示是Control消息，Control消息用来支持事务功能。
* last offset delta：RecordBatch中最后一个Record的offset与first offset的差值。主要被broker用来确认RecordBatch中Records的组装正确性。
* first timestamp：RecordBatch中第一条Record的时间戳。
* max timestamp：RecordBatch中最大的时间戳，一般情况下是指最后一个Record的时间戳，和last offset delta的作用一样，用来确保消息组装的正确性。
* producer id：用来支持幂等性，详细可以参考KIP-98。
* producer epoch：和producer id一样，用来支持幂等性。
* first sequence：和producer id、producer epoch一样，用来支持幂等性。
* records count：RecordBatch中Record的个数。

</details>

<details>

<summary>Record使用了大量varints，可以通过具体值来确定需要字节保存。去掉src字段，字段如下：</summary>

* length：消息总长度
* attributes：弃用，但是占据1B，以备未来扩展
* timestamp delta：时间戳增量，通过timestamp需要8字节，使用与RecordBatch的起始时间戳差值，可以节省占用字节数
* offset delta：位移增量
* headers：支持应用级扩展，不需要像v0和v1一样不得不将应用级属性嵌入消息体。一个Record可以包含0至多个Header

</details>

## 日志索引

Kafka的索引文件以稀疏索引的方式构建消息的索引，并不保证每个消息在索引文件中都有对应的索引项。

稀疏索引通过MappedByteBuffer将索引文件映射到内存里。偏移量索引文件及时间戳索引文件都是单调递增的，通过二分查找定位到位置或者小于指定值的最大值，然后通过拿到的物理地址定位日志文件位置。

### 偏移量索引

偏移量索引格式如下

relativeOffset 4B ：position 4B

* **relativeOffset**：相对偏移量，表示消息相对于baseOffset的偏移量，占4字节，当前文件名为baseOffset值
* **position**：物理地址，消息在日志分段文件中对应的物理地址，占4字节。

<mark style="color:red;">**注意**</mark>：kafka强制要求索引文件必须是索引项大小的整数倍，对偏移量索引文件而言，必须是8的整数倍。

### 时间戳索引

时间戳索引格式如下

timestamp 8B ：relativeOffset 4B

* **timestamp**：当前日志分段最大的时间戳
* **relativeOffset**：时间戳所对应的消息的相对偏移量

broker端参数`log.message.timestamp.type`的值为LogAppendTime，则消息时间戳必定能单调递增；如果是CreateTime类型则无法保证。

<mark style="color:red;">**注意**</mark>：与偏移量索引文件相似，索引文件大小必须是12的整数倍。

## 日志清理

Kafka 提供了两种日志清理策略：

1. **日志删除（Log Retention）** ：按照一定的保留策略直接删除不符合条件的日志分段；
2. **日志压缩（Log Compaction）** ：针对每个消息的 key 进行整合，对于有相同 key 的不同 value 值，只保留最后一个版本。

清理策略配置为 broker 端参数 `log.cleanup.policy`，共三种设置方式，即：

* <mark style="color:blue;">`delete`</mark>：即采用日志删除的清理策略，该设置为默认设置；
* <mark style="color:blue;">`compact`</mark>：采用日志压缩的清理策略（还需要将 log.cleaner.enable（默认值为 true）设定为 true。）；
* <mark style="color:blue;">`delete，compact`</mark>：同时支持日志删除和日志压缩两种策略；

### 日志删除

Kafka的日志管理器中有一个日志删除任务来周期性地检测和删除不符合保留条件的日志分段文件，这个周期可以通过 broker 端参数 `log.retention.check.interval.ms` 来配置，默认值为 300000，即 5 分钟。

当前日志分段的保留策略有 `3` 种：**基于时间的保留策略**、**基于日志大小的保留策略** 和 **基于日志起始偏移量的保留策略**。

#### 基于时间

日志删除任务会检查当前日志文件中是否有保留时间超过设定的阈值（retentionMs）的日志分段文件集合。retentionMs 可以通过 broker 端参数 `log.retention.hours`、`log.retention.minutes` 和 `log.retention.ms` 来配置，默认保留时间7天。

日志查找是根据<mark style="color:red;">日志分段中最大的时间戳 largestTimeStamp</mark>来计算。日志分段中的最大时间戳 largestTimeStamp的值，先查询该日志分段所对应的时间戳索引文件，查找时间戳索引文件中最后一条索引项，若最后一条索引项的时间戳字段值大于 0，则取其值，否则设置为最近修改时间 lastModifiedTime

删除日志分段时，首先会从 Log 对象中所维护日志分段的跳跃表中移除待删除的日志分段，以保证没有线程对这些日志分段进行读取操作。然后将日志分段所对应的所有文件添加上 `.deleted` 的后缀（包括对应的索引文件）。最后交由一个以“delete-file”命名的延迟任务来删除这些以 `.deleted`为后缀的文件，这个任务的延迟执行时间可以通过 `file.delete.delay.ms` 参数来调配，此参数的默认值为 60000，即 1 分钟。

#### 基于日志大小

日志删除任务会检查当前日志的大小是否超过设定的阈值（retentionSize）来寻找可删除的日志分段的文件集合（deletableSegments），retentionSize 可以通过 broker 端参数 `log.retention.bytes` 来配置，默认值为 -1，表示无穷大。

<mark style="color:red;">**注意**</mark>**： log.retention.bytes 配置的是 Log 中所有日志文件的总大小，而不是单个日志分段（`.log` 日志文件）的大小**。单个日志分段的大小由 broker 端参数 `log.segment.bytes` 来限制，默认值为 1073741824，即 1GB。

基于日志大小的保留策略与基于时间的保留策略类似，首先计算日志文件的总大小 size 和 retentionSize 的差值 diff，即计算需要删除的日志总大小，然后从日志文件中的第一个日志分段开始进行查找可删除的日志分段的文件集合 deletableSegments。查找出 deletableSegments 之后就执行删除操作，这个删除操作和基于时间的保留策略的删除操作相同。

#### 基于日志起始偏移量

判断某日志分段的**下一个日志分段**的起始偏移量 baseOffset 是否小于等于 logStartOffset，若是，则可以删除此日志分段。

日志的查找动作为从头开始遍历每个日志分段，直到找到所有的可删除的文件集合；删除操作同其他两个策略；

### 日志压缩

Kafka 中的 Log Compaction 是指在默认的日志删除（Log Retention）规则之外提供的一种清理过时数据的方式。Log Compaction 对于有相同 key 的不同 value 值，只保留最后一个版本。如果应用只关心 key 对应的最新 value 值，则可以开启 Kafka 的日志清理功能， Kafka 会定期将相同 key 的消息进行合并，只保留最新的 value 值。

Log Compaction 执行前后，日志分段中的每条消息的偏移量和写入时的偏移量保持一致。Log Compaction 会生成新的日志分段文件，日志分段中每条消息的物理位置会重新按照新文件来组织。Log Compaction 执行过后的偏移量不再是连续的，不过这并不影响日志的查询。

**注意 Log Compaction 是针对 key 的，所以在使用时应注意每个消息的 key 值不为 null。**

每个 broker 会启动 log.cleaner.thread（默认值为 1）个日志清理线程负责执行清理任务，这些线程会选择“污浊率”最高的日志文件进行清理。计算方式为：`污浊率=清理量/(清理量+污浊量)`

为了防止日志不必要的频繁清理操作，Kafka 还使用了参数 log.cleaner.min.cleanable.ratio（默认值为 0.5）来限定可进行清理操作的最小污浊率。

那么 Kafka 是怎么对日志文件中消息的 key 进行筛选操作呢？Kafka 中的每个日志清理线程会使用一个名为 `SkimpyOffsetMap` 的对象来构建 key 与 offset 的映射关系的哈希表。日志清理需要遍历两次日志文件，第一次遍历把每个 key 的哈希值和最后出现的 offset 都保存在 SkimpyOffsetMap 中，第二次遍历会检查每个消息是否符合保留条件，如果符合就保留下来，否则就会被清理。

默认情况下，SkimpyOffsetMap 使用 MD5 来计算 key 的哈希值，占用空间大小为 16B，根据这个哈希值来从 SkimpyOffsetMap 中找到对应的槽位，如果发生冲突则用 **线性探测法** 处理。

## 磁盘存储

磁盘的顺序写入速度远远高于随机写入速度，顺序写盘的速度不仅比随机写盘的速度快，而且也比随机写内存的速度快。操作系统还可以针对线性读写做深层次的优化，比如 **预读（read-ahead，提前将一个比较大的磁盘块读入内存）**  和 **后写（write-behind，将很多小的逻辑写操作合并起来组成一个大的物理写操作）**  技术。

### 页缓存

页缓存是操作系统实现的一种主要的磁盘缓存，以此用来减少对磁盘 I/O 的操作。把磁盘中的数据缓存到内存中，把对磁盘的访问变为对内存的访问。

* 当一个进程准备读取磁盘上的文件内容时，操作系统会先查看待读取的数据所在的页（page）是否在页缓存（pagecache）中，如果存在（命中）则直接返回数据，从而避免了对物理磁盘的 I/O 操作；如果没有命中，则操作系统会向磁盘发起读取请求并将读取的数据页存入页缓存，之后再将数据返回给进程。
* 同样，如果一个进程需要将数据写入磁盘，那么操作系统也会检测数据对应的页是否在页缓存中，如果不存在，则会先在页缓存中添加相应的页，**最后将数据写入对应的页。被修改过后的页也就变成了脏页，操作系统会在合适的时间把脏页中的数据写入磁盘，以保持数据的一致性**。

#### 1. 不必缓存两份数据

对一个进程而言，它会在进程内部缓存处理所需的数据，然而这些数据有可能还缓存在操作系统的页缓存中，因此同一份数据有可能被缓存了两次。

并且，除非使用 Direct I/O 的方式，否则页缓存很难被禁止。

#### 2. 减少垃圾回收

此外，Java 一般对象的内存开销非常大，通常会是真实数据大小的几倍甚至更多，空间使用率低下；Java 的垃圾回收会随着堆内数据的增多而变得越来越慢。

基于这些因素，使用文件系统并依赖于页缓存的做法明显要优于维护一个进程内缓存或其他结构，省去了一份进程内部的缓存消耗，同时还可以通过结构紧凑的字节码来替代使用对象的方式以节省更多的空间。如此，我们可以在 32GB 的机器上使用 28GB 至 30GB 的内存而不用担心 GC 所带来的性能问题。

#### 3. 减少进程内缓存重建

此外，即使 Kafka 服务重启，页缓存还是会保持有效，然而进程内的缓存却需要重建。这样也简化了代码逻辑，因为维护页缓存和文件之间的一致性交由操作系统来负责，这样会比进程内维护更加安全有效。

### 零拷贝

**零拷贝是指将数据直接从磁盘文件复制到网卡设备中，而不需要经由应用程序之手。零拷贝大大提高了应用程序的性能，减少了内核和用户模式之间的上下文切换。对 Linux 操作系统而言，零拷贝技术依赖于底层的 `sendfile()` 方法实现。**（在java中对`FileChannal.transferTo()`底层就是`sendfile()`方法）

#### 正常情况

正常情况下，将磁盘消息传递出去需要以下步骤：

`read(file, tmp_buf, len)`

`write(socket, tmp_buf, len)`

1. 调用`read()`将磁盘内容复制到内核的 **Read Buffer** 中
2. cpu将内核数据复制到用户模式下应用程序内
3. 调用`write()`将用户模式内容复制到内核的 **Socket Buffer** 中
4. 将内核的数据复制到网卡设备中传送

经历了4次复制，4次内核和用户模式的上下文切换

#### 使用零拷贝

零拷贝技术通过 DMA（Direct Memory Access）技术将文件内容复制到内核模式下的 **Read Buffer** 中。不过没有数据被复制到 Socket Buffer，只有包含数据的位置和长度的信息的 **`文件描述符`** 被加到 Socket Buffer 中。DMA 引擎直接将数据从内核模式中传递到网卡设备（协议引擎）。

这样数据只经历了 2 次复制就从磁盘中传送出去了，并且上下文切换也变成了 2 次。`零拷贝是针对内核模式而言的，数据在内核模式下实现了零拷贝`。

### 磁盘IO（扩展知识）
