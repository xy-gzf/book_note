---
description: 应用
---

# Apply

## cache coherence（缓存一致性）

#### 先删缓存在更新数据库

线程A删除缓存后，还未来得及更新数据库，b读取数据发现缓存没有读完数据库并塞回缓存，线程A更新数据库。此时缓存不一致。

:thumbsup:线程A更新完数据库的值后，可以让它sleep一小段时间，再进行一次缓存删除操作。"延时双删"

#### 先更新数据库在删除缓存

如果线程A更新了数据库中的值，但还没来得及删除缓存中的值，线程B这时候开始读取数据，此时，线程B查询缓存时，命中了缓存，就会直接使用缓存中的值，该值为旧值。

:thumbsup:如果并发请求量不高的话，其实基本上不会有线程读到旧值。这种情况总体对业务影响较小。一般在生产环境中，也推荐采用该模式。

#### 重试机制

可以把要删除的缓存值或者要更新的数据库的值放到消息队列中，当应用没能够成功地删除缓存或者是更新数据库的值的时候，可以从消息队列中消费这些值，这里消费消息队列的服务叫job，然后再次进行删除或者更新，起到一个兜底补偿的作用，以此来保证最终的一致性。

如果能够成功地删除或更新，就需要把这些值从消息队列中去除，以免重复操作，此时，我们也可以保证数据库和缓存数据的一致了，否则的话，我们还需要再次进行重试，如果重试超过一定次数还是失败，这时候一般都需要记录错误日志或者发送告警通知。

#### 并发读写

A读缓存，未命中读数据库，此时B更新数据库，并更新缓存，然后A将数据库读到的值更新缓存，导致缓存不一致

:thumbsup:提高更新数据库后的更新缓存优先级，降低读数据库后回塞缓存的优先级。（读数据库后set cache改为add cache，add cache即setnx操作）

## distributed locks（分布式锁）

#### 简单实现 <a href="#jian-dan-shi-xian" id="jian-dan-shi-xian"></a>

Comment

* 设置使用setnx，成功设置则加上了锁，释放锁正常使用del。Comment
* 避免出现死锁情况，则需要给定一个过期时间，因此出现了setexnx命令。Comment
* 如果被其他服务器释放锁，因此set时值需要为服务器标识，是自己的才可以进行delComment
* 查看和del不是原子性操作，所以需要使用lua脚本保证原子性Comment
* 过期时间问题，需要开启守护线程不断进行续期Comment

Comment至此，单机redis实现分布式锁基本可以满足大多数情况。Comment

#### 小结 <a href="#xiao-jie" id="xiao-jie"></a>

Comment基于 Redis 的实现分布式锁，遇到的问题以及对应的解决方案：Comment

* **死锁**：设置过期时间Comment
* **过期时间评估不好，锁提前过期**：守护线程，自动续期Comment
* **锁被别人释放**：锁写入唯一标识，释放锁先检查标识，再释放Comment

Comment​Comment**发生主从切换时候，分布式锁依旧安全吗？**&#x43;omment不一定，看如下场景：Comment

1. 客户端 1 在主库上执行 SET 命令，加锁成功Comment
2. 此时主库异常宕机，SET 命令还未同步到从库上（主从复制是异步的）Comment
3. 从库被哨兵提升为新主库，这个锁在新的主库上，未被同步到，所以丢失！Comment

Comment因此这样实现在引入 Redis 副本后，分布锁还是可能会受到影响。Comment

#### 那么Redis集群如何实现分布式锁呢？ <a href="#na-me-redis-ji-qun-ru-he-shi-xian-fen-bu-shi-suo-ne" id="na-me-redis-ji-qun-ru-he-shi-xian-fen-bu-shi-suo-ne"></a>

## bloom filter（布隆过滤器）

#### 什么是布隆过滤器（Bloom Filter）？

布隆过滤器由 Burton Howard Bloom 于 1970 年提出，用于判断一个元素是否在集合中。&#x20;

布隆过滤器（Bloom filter）是一种非常节省空间的概率数据结构（space-efficient probabilistic data structure），运行速度快（时间效率），占用内存小（空间效率），但是有一定的误判率且无法删除元素。本质上由一个很长的二进制向量和一系列随机映射函数组成。

[🔗](https://llimllib.github.io/bloomfilter-tutorial/zh_CN/)

#### 布隆过滤器有什么特性

* 检查一个元素是否在集成中；&#x20;
* 检查结果分为 2 种：一定不在集合中、可能在集合中；&#x20;
* 布隆过滤器支持添加元素、检查元素，但是不支持删除元素；&#x20;
* 检查结果的 “可能在集合中” 说明存在一定误判率；&#x20;
* 已经添加进入布隆过滤器的元素是不会被误判的，仅未添加过的元素才可能被误判；&#x20;
* 相比 set、Bitmaps 非常节省空间：因为只存储了指纹信息，没有存储元素本身；
* 添加的元素超过预设容量越多，误报的可能性越大。

### Redis如何实现bloom filter

#### 开发中一种简单的方式

通过语言的相关库来构造一个bf对象，通过将该对象进行序列化反序列化存入redis的string内。每次取出进行判段，添加后塞入redis内

#### **优点：**

* 操作快速，容易实现，易理解

#### **缺点：**

* 序列化反序列化开销较大，对cpu负载较高。（容量较大的bf，不建议使用）

eg：go语言`github.com/willf/bloom`

布隆过滤器本就是为了高并发时过滤掉大量的无效请求，但是高并发情况下序列化反序列化开销较大，对cpu负载较高，因此不要轻易使用该方式。

#### 使用Redis的bit位实现

通过redis的bit位来实现，不需要库那么大的cpu开销，不需要序列化与反序列化。

```go
/*
 * following 布隆过滤器
 * @key 布隆过滤器的key
 * @element 要判断的元素
 * @filterSize 布隆过滤器的大小
 * @hashCount  哈希函数的个数
 * 暂时不支持动态扩展布隆过滤器的大小，所以同一个布隆过滤器的大小和哈希函数个数要保持一致
 */

// BloomFilterAdd 方法，将元素添加到布隆过滤器中
func (c *Cache) BloomFilterAdd(key, element string, filterSize, hashCount uint) error {
	for i := uint(0); i < hashCount; i++ {
		err := c.SetBit(key, uint32(hash(element, i)%filterSize), 1)
		if err != nil {
			return err
		}
	}
	return nil
}

// BloomFilterIsExist 方法，判断元素是否在布隆过滤器中存在
func (c *Cache) BloomFilterIsExist(key, element string, filterSize, hashCount uint) bool {
	for i := uint(0); i < hashCount; i++ {
		if res, _ := c.GetBit(key, uint32(hash(element, i)%filterSize)); !res {
			return false
		}
	}
	return true
}

// hash 函数，自定义哈希函数
func hash(s string, seed uint) uint {
	var h uint = seed
	for i := 0; i < len(s); i++ {
		h = h*31 + uint(s[i])
	}
	return h
}
```

#### **步骤：**

1. 生命哈希函数，可以将数据映射到bit位的某一位上
2. 指定布隆过滤器大小和哈希函数个数
3. 添加时将数据通过哈希函数个数轮哈希值塞入bit内
4. 判断存在时判断是否都为1

#### **优点：**

* 方便、快捷、无需序列化

#### **缺点：**

* 需要通过哈希函数实现及bf大小来控制误判率
* 无法动态扩容

### 生产中使用方式

Redis4.0以插件的形式提供布隆过滤器

安装指引文档：[🔗](https://redis.io/docs/stack/bloom/quick_start/)

线上环境可以使用redis企业版

#### **特性：**

* 内存占用低。&#x20;
* 可动态扩容。&#x20;
* 可自定义的误判率（False Positive Rate）且在扩容时保持不变。

#### Scalable-Bloom-Filter扩容过程

该SBF一共包含BF0和BF1两层。在一开始，SBF只包含BF0层，假设在插入a、b、c三个元素后，BF0层已经无法保证用户设定的误判率，此时会创建新的一层（BF1层）进行扩容。因此，后面的d、e、f元素会插入到BF1层中。同理，当BF1层也无法满足误判率时，会创建新的一层（BF2层），如此进行下去。

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsmBJIuFrw1cRlevdXezN%2Fuploads%2FdfKJXUkzRRj3XaoMK18y%2Fimage.png?alt=media&#x26;token=3e56936d-d508-4590-830b-e4c11ec0b1dc" alt=""><figcaption><p>bf</p></figcaption></figure>

Scalable Bloom Filter只会向最后一层插入数据，同时也从最后一层开始查询，直到查询至BF0层。

#### 注意事项

* 虽然TairBloom支持扩容，但在实际使用过程中请避免发生扩容操作，建议将该功能视为保障措施，若实际容量超过预设容量时，TairBloom能通过扩容操作，保障业务正常写入，规避线上事故。&#x20;
* 查找的时间复杂度为O(log N), N为redis bloom filter的层数，所以，如果不限制扩容次数，会导致查询效率下降。
* &#x20;发生扩容时，新建的一层容量为上层的2倍，在设计BF时，需要考虑扩容几次时会导致oom。

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FsmBJIuFrw1cRlevdXezN%2Fuploads%2F45zNkikVS69Xs2ayWa5o%2Fimage.png?alt=media&#x26;token=a194916c-c215-4937-8493-7cdf4c07d354" alt=""><figcaption><p>notion</p></figcaption></figure>

#### 优点：

* 使用简单
* 功能强大、支持动态扩容

#### 缺点：

* 企业级redis价格偏贵（因此在企业级redis上最好只使用一般redis不支持的功能）
