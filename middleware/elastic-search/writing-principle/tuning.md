---
description: 写入性能调优
---

# tuning

### 基本原则 <a href="#usercontent31-ji-ben-yuan-ze" id="usercontent31-ji-ben-yuan-ze"></a>

写性能调优是建立在对 Elasticsearch 的写入原理之上。ES 数据写入具有一定的延时性，这是为了减少频繁的索引文件产生。默认情况下 ES 每秒生成一个 segment 文件，当达到一定阈值的时候 会执行merge，merge 过程发生在 JVM中，频繁的生成 Segmen 文件可能会导致频繁的触发 FGC，导致 OOM。为了避免避免这种情况，通常采取的手段是降低 segment 文件的生成频率，手段有两个，一个是 增加时间阈值，另一个是增大 Buffer的空间阈值，因为缓冲区写满也会生成 Segment 文件。

生产经常面临的写入可以分为两种情况：

**高频低量**：高频的创建或更新索引或文档一般发生在 处理 C 端业务的场景下。

**低频高量**：一般情况为定期重建索引或批量更新文档数据。

在搜索引擎的业务场景下，用户一般并不需要那么高的写入实时性。比如你在网站发布一条征婚信息，或者二手交易平台发布一个商品信息。其他人并不是马上能搜索到的，这其实也是正常的处理逻辑。这个延时的过程需要处理很多事情，业务层面比如：你的信息需要后台审核。你发布的内容在搜索服务中需要建立索引，而且你的数据可能并不会马上被写入索引，而是等待要写入的数据达到一定数量之后，批量写入。这种操作优点类似于我们快递物流的场景，只有当快递数量达到一定量级的时候，比如能装满整个车的时候，快递车才会发车。因为反正是要跑一趟，装的越多，平均成本越低。这和我们数据写入到磁盘的过程是非常相似的，我们可以把一条文档数据看做是一个快递，而快递车每次发车就是向磁盘写入数据的一个过程。这个过程不宜太多，太多只会降低性能，就是体现在运输成本上面。而对于我们数据写入而言就是体现在我们硬件性能损耗上面。

### 优化手段 <a href="#usercontent32-you-hua-shou-duan" id="usercontent32-you-hua-shou-duan"></a>

以下为常见 数据写入的调优手段，写入调优均以提升写入吞吐量和并发能力为目标，而非提升写入实时性。

#### 增加 flush 时间间隔 <a href="#usercontent321-zeng-jia-flush-shi-jian-jian-ge" id="usercontent321-zeng-jia-flush-shi-jian-jian-ge"></a>

目的是减小数据写入磁盘的频率，减小磁盘IO频率。

#### 增加refresh\_interval的参数值 <a href="#usercontent322-zeng-jia-refreshinterval-de-can-shu-zhi" id="usercontent322-zeng-jia-refreshinterval-de-can-shu-zhi"></a>

目的是减少segment文件的创建，减少segment的merge次数，merge是发生在jvm中的，有可能导致full GC，增加refresh会降低搜索的实时性。

ES的 refresh 行为非常昂贵，并且在正在进行的索引活动时经常调用，会降低索引速度，这一点在索引写入原理中介绍过，了解索引的写入原理，可以关注我的博客Elastic开源社区。

默认情况下，Elasticsearch 每秒定期刷新索引，但仅在最近 30 秒内收到一个或多个搜索请求的索引上。

如果没有搜索流量或搜索流量很少（例如每 5 分钟不到一个搜索请求）并且想要优化索引速度，这是最佳配置。此行为旨在在不执行搜索的默认情况下自动优化批量索引。建议显式配置此配置项，如 30秒。

#### 增加Buffer大小 <a href="#usercontent323-zeng-jia-buffer-da-xiao" id="usercontent323-zeng-jia-buffer-da-xiao"></a>

本质也是减小refresh的时间间隔，因为导致segment文件创建的原因不仅有时间阈值，还有buffer空间大小，写满了也会创建。 默认最小值 48MB< 默认值 JVM 空间的10% < 默认最大无限制

#### 关闭副本 <a href="#usercontent324-guan-bi-fu-ben" id="usercontent324-guan-bi-fu-ben"></a>

当需要单次写入大量数据的时候，建议关闭副本，暂停搜索服务，或选择在检索请求量谷值区间时间段来完成。

第一，是减小读写之间的资源抢占，读写分离 第二，当检索请求数量很少的时候，可以减少甚至完全删除副本分片，关闭segment的自动创建以达到高效利用内存的目的，因为副本的存在会导致主从之间频繁的进行数据同步，大大增加服务器的资源占用。 具体可通过则设置index.number\_of\_replicas 为0以加快索引速度。没有副本意味着丢失单个节点可能会导致数据丢失，因此数据保存在其他地方很重要，以便在出现问题时可以重试初始加载。初始加载完成后，可以设置index.number\_of\_replicas改回其原始值。

#### 禁用swap <a href="#usercontent325-jin-yong-swap" id="usercontent325-jin-yong-swap"></a>

大多数操作系统尝试将尽可能多的内存用于文件系统缓存，并急切地换掉未使用的应用程序内存。这可能导致部分 JVM 堆甚至其可执行页面被换出到磁盘。

交换对性能和节点稳定性非常不利，应该不惜一切代价避免。它可能导致垃圾收集持续几分钟而不是几毫秒，并且可能导致节点响应缓慢甚至与集群断开连接。在Elastic分布式系统中，让操作系统杀死节点更有效。

#### 使用多个工作线程 <a href="#usercontent326-shi-yong-duo-ge-gong-zuo-xian-cheng" id="usercontent326-shi-yong-duo-ge-gong-zuo-xian-cheng"></a>

发送批量请求的单个线程不太可能最大化 Elasticsearch 集群的索引容量。为了使用集群的所有资源，应该从多个线程或进程发送数据。除了更好地利用集群的资源外，还有助于降低每个 fsync 的成本。

确保注意 TOO\_MANY\_REQUESTS (429)响应代码（EsRejectedExecutionException使用 Java 客户端），这是 Elasticsearch 告诉我们它无法跟上当前索引速度的方式。发生这种情况时，应该在重试之前暂停索引，最好使用随机指数退避。

与调整批量请求的大小类似，只有测试才能确定最佳工作线程数量是多少。这可以通过逐渐增加线程数量来测试，直到集群上的 I/O 或 CPU 饱和。

#### 避免使用稀疏数据 <a href="#usercontent327-bi-mian-shi-yong-xi-shu-shu-ju" id="usercontent327-bi-mian-shi-yong-xi-shu-shu-ju"></a>

#### max\_result\_window参数 <a href="#usercontent328maxresultwindow-can-shu" id="usercontent328maxresultwindow-can-shu"></a>

max\_result\_window是分页返回的最大数值，默认值为10000。max\_result\_window本身是对JVM的一种保护机制，通过设定一个合理的阈值，避免初学者分页查询时由于单页数据过大而导致OOM。

在很多业务场景中经常需要查询10000条以后的数据，当遇到不能查询10000条以后的数据的问题之后，网上的很多答案会告诉你可以通过放开这个参数的限制，将其配置为100万，甚至1000万就行。但是如果仅仅放开这个参数就行，那么这个参数限制的意义有何在呢？如果你不知道这个参数的意义，很可能导致的后果就是频繁的发生OOM而且很难找到原因，设置一个合理的大小是需要通过你的各项指标参数来衡量确定的，比如你用户量、数据量、物理内存的大小、分片的数量等等。通过监控数据和分析各项指标从而确定一个最佳值，并非越大越好

## 查询调优 <a href="#usercontent4-cha-xun-tiao-you" id="usercontent4-cha-xun-tiao-you"></a>

### 读写性能不可兼得 <a href="#usercontent41-du-xie-xing-neng-bu-ke-jian-de" id="usercontent41-du-xie-xing-neng-bu-ke-jian-de"></a>

首先要明确一点：鱼和熊掌不可兼得。读写性能调优在很多场景下是只能二选一的。牺牲 A 换 B 的行为非常常见。索引本质上也是通过空间换取时间。写生写入实时性就是为了提高检索的性能。

当你在二手平台或者某垂直信息网站发布信息之后，是允许有信息写入的延时性的。但是检索不行，甚至 1 秒的等待时间对用户来说都是无法接受的。满足用户的要求甚至必须做到10 ms以内。

### 优化手段 <a href="#usercontent42-you-hua-shou-duan" id="usercontent42-you-hua-shou-duan"></a>

#### 避免单次召回大量数据 <a href="#usercontent421-bi-mian-dan-ci-zhao-hui-da-liang-shu-ju" id="usercontent421-bi-mian-dan-ci-zhao-hui-da-liang-shu-ju"></a>

搜索引擎最擅长的事情是从海量数据中查询少量相关文档，而非单次检索大量文档。非常不建议动辄查询上万数据。如果有这样的需求，建议使用滚动查询

#### 避免单个文档过大 <a href="#usercontent422-bi-mian-dan-ge-wen-dang-guo-da" id="usercontent422-bi-mian-dan-ge-wen-dang-guo-da"></a>

鉴于默认http.max\_content\_length设置为 100MB，Elasticsearch 将拒绝索引任何大于该值的文档。您可能决定增加该特定设置，但 Lucene 仍然有大约 2GB 的限制。

即使不考虑硬性限制，大型文档通常也不实用。大型文档对网络、内存使用和磁盘造成了更大的压力，即使对于不请求的搜索请求也是如此，\_source因为 Elasticsearch\_id在所有情况下都需要获取文档的文件系统缓存有效。对该文档进行索引可能会占用文档原始大小的倍数的内存量。Proximity Search（例如短语查询）和高亮查询也变得更加昂贵，因为它们的成本直接取决于原始文档的大小。

有时重新考虑信息单元应该是什么是有用的。例如，您想让书籍可搜索的事实并不一定意味着文档应该包含整本书。使用章节甚至段落作为文档可能是一个更好的主意，然后在这些文档中拥有一个属性来标识它们属于哪本书。这不仅避免了大文档的问题，还使搜索体验更好。例如，如果用户搜索两个单词fooand bar，则不同章节之间的匹配可能很差，而同一段落中的匹配可能很好。

#### 单次查询10条文档 好于 10次查询每次一条 <a href="#usercontent423-dan-ci-cha-xun-10-tiao-wen-dang-hao-yu-10-ci-cha-xun-mei-ci-yi-tiao" id="usercontent423-dan-ci-cha-xun-10-tiao-wen-dang-hao-yu-10-ci-cha-xun-mei-ci-yi-tiao"></a>

批量请求将产生比单文档索引请求更好的性能。但是每次查询多少文档最佳，不同的集群最佳值可能不同，为了获得批量请求的最佳阈值，建议在具有单个分片的单个节点上运行基准测试。首先尝试一次索引 100 个文档，然后是 200 个，然后是 400 个等。在每次基准测试运行中，批量请求中的文档数量翻倍。当索引速度开始趋于平稳时，就可以获得已达到数据批量请求的最佳大小。在相同性能的情况下，当大量请求同时发送时，太大的批量请求可能会使集群承受内存压力，因此建议避免每个请求超过几十兆字节。

#### 数据建模 <a href="#usercontent424-shu-ju-jian-mo" id="usercontent424-shu-ju-jian-mo"></a>

很多人会忽略对 Elasticsearch 数据建模的重要性。

nested属于object类型的一种，是Elasticsearch中用于复杂类型对象数组的索引操作。Elasticsearch没有内部对象的概念，因此，ES在存储复杂类型的时候会把对象的复杂层次结果扁平化为一个键值对列表。

特别是，应避免连接。Nested 可以使查询慢几倍，Join 会使查询慢数百倍。两种类型的使用场景应该是：Nested针对字段值为非基本数据类型的时候，而Join则用于 当子文档数量级非常大的时候。

关于数据建模，在我的博客中有详细的讲解，此处不再赘述

#### 给系统留足够的内存 <a href="#usercontent425-gei-xi-tong-liu-zu-gou-de-nei-cun" id="usercontent425-gei-xi-tong-liu-zu-gou-de-nei-cun"></a>

Lucene的数据的fsync是发生在OS cache的，要给OS cache预留足够的内从大小，详见JVM调优。

#### 预索引 <a href="#usercontent426-yu-suo-yin" id="usercontent426-yu-suo-yin"></a>

利用查询中的模式来优化数据的索引方式。例如，如果所有文档都有一个price字段，并且大多数查询 range 在固定的范围列表上运行聚合，可以通过将范围预先索引到索引中并使用聚合来加快聚合速度。

#### 使用 filter 代替 query <a href="#usercontent427-shi-yong-filter-dai-ti-query" id="usercontent427-shi-yong-filter-dai-ti-query"></a>

query和filter的主要区别在： filter是结果导向的而query是过程导向。query倾向于“当前文档和查询的语句的相关度”而filter倾向于“当前文档和查询的条件是不是相符”。即在查询过程中，query是要对查询的每个结果计算相关性得分的，而filter不会。另外filter有相应的缓存机制，可以提高查询效率。

#### 避免深度分页 <a href="#usercontent428-bi-mian-shen-du-fen-ye" id="usercontent428-bi-mian-shen-du-fen-ye"></a>

避免单页数据过大，可以参考百度或者淘宝的做法。es提供两种解决方案 scroll search 和 search after。关于深度分页的详细原理，推荐阅读：详解Elasticsearch深度分页问题

#### 使用 Keyword 类型 <a href="#usercontent429-shi-yong-keyword-lei-xing" id="usercontent429-shi-yong-keyword-lei-xing"></a>

并非所有数值数据都应映射为数值字段数据类型。Elasticsearch为 查询优化数字字段，例如integeror long。如果不需要范围查找，对于 term查询而言，keyword 比 integer 性能更好。

#### 避免使用脚本 <a href="#usercontent4210-bi-mian-shi-yong-jiao-ben" id="usercontent4210-bi-mian-shi-yong-jiao-ben"></a>

Scripting是Elasticsearch支持的一种专门用于复杂场景下支持自定义编程的强大的脚本功能。相对于 DSL 而言，脚本的性能更差，DSL能解决 80% 以上的查询需求，如非必须，尽量避免使用 Script
