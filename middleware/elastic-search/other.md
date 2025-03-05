---
description: 其他
---

# Other

## suggester

### term suggester

只基于tokenizer之后的单个term去匹配建议词，并不会考虑多个term之间的关系

```json
POST <index>/_search
{ 
  "suggest": {
    "<suggest_name>": {
      "text": "<search_content>",
      "term": {
        "suggest_mode": "<suggest_mode>",
        "field": "<field_name>"
      }
    }
  }
}
```

#### **Options：**

* **text**：用户搜索的文本
* **field**：要从哪个字段选取推荐数据
* **analyzer**：使用哪种分词器
* **size**：每个建议返回的最大结果数
* **sort**：如何按照提示词项排序，参数值只可以是以下两个枚举：
  * **score**：分数>词频>词项本身
  * **frequency**：词频>分数>词项本身
* **suggest\_mode**：搜索推荐的推荐模式，参数值亦是枚举：
  * missing：默认值，仅为不在索引中的词项生成建议词
  * popular：仅返回与搜索词文档词频或文档词频更高的建议词
  * always：根据 建议文本中的词项 推荐 任何匹配的建议词
* **max\_edits**：可以具有最大偏移距离候选建议以便被认为是建议。只能是1到2之间的值。任何其他值都将导致引发错误的请求错误。默认为2
* **prefix\_length**：前缀匹配的时候，必须满足的最少字符
* **min\_word\_length**：最少包含的单词数量
* **min\_doc\_freq**：最少的文档频率
* **max\_term\_freq**：最大的词频

### phrase suggester

phrase suggester和term suggester相比，对建议的文本会参考上下文，也就是一个句子的其他token，不只是单纯的token距离匹配，它可以基于共生和频率选出更好的建议。

```sh
POST <index>/_search
{ 
  "suggest": {
    "text": "<search_content>",
    "simple_phrase": {
      "field": "<field_name>"
    }
  }
}
```

#### Notion:

* 需要mapping指定格式

#### Options：

* real\_word\_error\_likelihood： 此选项的默认值为 0.95。此选项告诉 Elasticsearch 索引中 5% 的术语拼写错误。这意味着随着这个参数的值越来越低，Elasticsearch 会将越来越多存在于索引中的术语视为拼写错误，即使它们是正确的
* max\_errors：为了形成更正，最多被认为是拼写错误的术语的最大百分比。默认值为 1
* confidence：默认值为 1.0，最大值也是。该值充当与建议分数相关的阈值。只有得分超过此值的建议才会显示。例如，置信度为 1.0 只会返回得分高于输入短语的建议
* collate：告诉 Elasticsearch 根据指定的查询检查每个建议，以修剪索引中不存在匹配文档的建议。在这种情况下，它是一个匹配查询。由于此查询是模板查询，因此搜索查询是当前建议，位于查询中的参数下。可以在查询下的“params”对象中添加更多字段。同样，当参数“prune”设置为true时，我们将在响应中增加一个字段“collate\_match”，指示建议结果中是否存在所有更正关键字的匹配
* direct\_generator：phrase suggester使用候选生成器生成给定文本中每个项可能的项的列表。单个候选生成器类似于为文本中的每个单独的调用term suggester。生成器的输出随后与建议候选项中的候选项结合打分。目前只支持一种候选生成器，即direct\_generator。建议API接受密钥直接生成器下的生成器列表；列表中的每个生成器都按原始文本中的每个项调用。

### **completion suggester**

**最常使用**

自动补全，自动完成，支持三种查询【前缀查询（prefix）模糊查询（fuzzy）正则表达式查询（regex)】 ，主要针对的应用场景就是"Auto Completion"。 此场景下用户每输入一个字符的时候，就需要即时发送一次查询请求到后端查找匹配项，在用户输入速度较高的情况下对后端响应速度要求比较苛刻。因此实现上它和前面两个Suggester采用了不同的数据结构，索引并非通过倒排来完成，而是将analyze过的数据编码成FST和索引一起存放。对于一个open状态的索引，FST会被ES整个装载到内存里的，进行前缀查找速度极快。但是FST只能用于前缀查找，这也是Completion Suggester的局限所在。

```sh
POST <index>/_search
{ 
  "suggest": {
    "<suggest_name>": {
      "prefix/regex": "<search_content>",
      "completion": {
        "field": "<field_name>",
        "fuzzy": {
        }
      }
    }
  }
}
```

#### Options：

* completion：es的一种特有类型，专门为suggest提供，基于内存，性能很高。
* prefix query：基于前缀查询的搜索提示，是最常用的一种搜索推荐查询。
  * prefix：客户端搜索词
  * field：建议词字段
  * size：需要返回的建议词数量（默认5）
  * skip\_duplicates：是否过滤掉重复建议，默认false
* fuzzy query
  * fuzziness：允许的偏移量，默认auto
  * transpositions：如果设置为true，则换位计为一次更改而不是两次更改，默认为true。
  * min\_length：返回模糊建议之前的最小输入长度，默认 3
  * prefix\_length：输入的最小长度（不检查模糊替代项）默认为 1
  * unicode\_aware：如果为true，则所有度量（如模糊编辑距离，换位和长度）均以Unicode代码点而不是以字节为单位。这比原始字节略慢，因此默认情况下将其设置为false。
* regex query：可以用正则表示前缀，不建议使用

### **context suggester**

完成建议者会考虑索引中的所有文档，但是通常来说，我们在进行智能推荐的时候最好通过某些条件过滤，并且有可能会针对某些特性提升权重。

```json
POST <index>/_search
{ 
  "suggest": {
    "<suggest_name>": {
      "prefix/regex": "<search_content>",
      "completion": {
        "field": "<field_name>",
        "contexts": {
          "place_type": []
        }
      }
    }
  }
}
```

#### Options：

* contexts：上下文对象，可以定义多个
  * name：`context`的名字，用于区分同一个索引中不同的`context`对象。需要在查询的时候指定当前name
  * type：`context`对象的类型，目前支持两种：category和geo，分别用于对suggest item分类和指定地理位置。
  * boost：权重值，用于提升排名
* path：如果没有path，相当于在PUT数据的时候需要指定context.name字段，如果在Mapping中指定了path，在PUT数据的时候就不需要了，因为 Mapping是一次性的，而PUT数据是频繁操作，这样就简化了代码。这段解释有木有很牛逼，网上搜到的都是官方文档的翻译，觉悟雷同。

## Deep paging

### ES中`from+size`分页 <a href="#usercontent11es-zhong-fromsize-fen-ye" id="usercontent11es-zhong-fromsize-fen-ye"></a>

```python
GET order_2290w/_search
{
  "from": 0,
  "size": 5
}

# 如果我们查询的数据页数特别大，当from + size大于10000的时候，就会出现问题，报错信息：
GET order_2290w/_search
{
  "from": 10000,
  "size": 5
}

"reason" : "Result window is too large, from + size must be less than or equal to:"+ 
"[10000] but was [10005]. See the scroll api for a more efficient way to request large "+
"data sets. This limit can be set by changing the [index.max_result_window] index level "+
"setting."
```

报错信息的解释为当前查询的结果超过了`10000`的最大值。那么疑问就来了，明明只查询了5条数据，为什么它计算最大值要加上我from的数量呢？而且Elasticsearch不是号称PB及数据秒级查询，几十亿的数据都没问题，怎么还限制最大查询前10000条数据呢？这里有一个字很关键：“`前`”，前10000条意味着什么？意味着数据肯定是按照某种顺序排列的，ES中如果不人工指定排序字段，那么最终结果将按照相关度评分排序。

分布式系统都面临着同一个问题，数据的排序不可能在同一个节点完成。一个简单的需求，比如：

#### 案例解释什么是深度分页 <a href="#usercontent12-an-li-jie-shi-shen-me-shi-shen-fen-ye" id="usercontent12-an-li-jie-shi-shen-me-shi-shen-fen-ye"></a>

**从`10万`名高考生中查询成绩为的`10001-10100`位的`100`名考生的信息。** 看似简单的查询其实并不简单，我们来画图解释一下：

<figure><img src="https://1301551370.vod-qcloud.com/3b912203vodsh1301551370/cb8cf8851253642696718210467/xIN29HNIa5sA.png" alt="" width="563"><figcaption><p>deep paging sort1</p></figcaption></figure>

索引数据在写入是并无法判断在执行业务查询时的具体排序规则，因此排序是随机的。而由于ES的分片和数据分配策略为了提高数据在检索时的准确度，会把数据尽可能均匀的分布在不同的分片。假设此时我们有五个分片，每个分片中承载`2万`条有效数据。按照需求我们需要去除成绩在`10001`到`10100`的一百名考生的信息，就要先按照成绩进行倒序排列。然后按照`page_size: 100`&`page_index: 101`进行查询。即查询按照成绩排序，第`101页`的100位学员信息。

单机数据库的查询逻辑很简单，先按照把10万学生成绩排序，然后从`前10100`条数据数据中取出第`10001-10100`条。即按照100为一页的第101页数据。

分布式数据库不同，学员成绩是被分散保存在每个分片中的，你无法保证要查询的这一百位学员的成绩一定都在某一个分片中，结果很有可能是存在于每个分片。解决办法是从每个分片中取出当前分片的`前10100`名学员成绩，然后汇总成`50500`条数据再次排序，然后从排序后的这`50500`个成绩中查询前`10100`的成绩，此时才能保证一定是整个索引中的成绩的前`10100`名。

### 深度分页危害及如何避免

#### 带来的问题

每次有序的查询都会在每个分片中执行单独的查询，然后进行数据的二次排序，而这个二次排序的过程是发生在heap中的，也就是说当你单次查询的数量越大，那么堆内存中汇总的数据也就越多，对内存的压力也就越大。

因此，如果查询的数据排序越靠后，就越容易导致OOM（Out Of Memory）情况的发生，频繁的深分页查询会导致频繁的FGC。 ES为了避免用户在不了解其内部原理的情况下而做出错误的操作，设置了一个阈值，即`max_result_window`，其默认值为`10000`，其作用是为了保护堆内存不被错误操作导致溢出。因此也就出现了文章一开始所演示的问题。

**`max_result_window`参数**

max\_result\_window是分页返回的最大数值，默认值为10000。max\_result\_window本身是对JVM的一种保护机制，通过设定一个合理的阈值，避免初学者分页查询时由于单页数据过大而导致OOM。

在很多业务场景中经常需要查询10000条以后的数据，当遇到不能查询10000条以后的数据的问题之后，网上的很多答案会告诉你可以通过放开这个参数的限制，将其配置为100万，甚至1000万就行。但是如果仅仅放开这个参数就行，那么这个参数限制的意义有何在呢？如果你不知道这个参数的意义，很可能导致的后果就是频繁的发生OOM而且很难找到原因，设置一个合理的大小是需要通过你的各项指标参数来衡量确定的，比如你用户量、数据量、物理内存的大小、分片的数量等等。通过监控数据和分析各项指标从而确定一个最佳值，并非越大约好。

### 解决方案

#### 1. 避免深度分页

限制搜索更深页数的数据

* 首先那些直接输入很大的页码，直接点击跳页的用户，本身就是恶意用户，阻止其行为是理所应当，因此删除“`跳页`”功能并无不妥
* 其次，真正的通过搜索引擎来检索其意向数据的用户，只关心前几页数据，即便他通过分页条跳了几页，但这种搜索并不涉及深度分页，即便它不停的点下去，也有其它方案解决此问题。
* 类似淘宝这种直接截断前100页数据的做法，看似暴力，其实是在保证用户体验的前提下，极大的提升了搜索的性能

#### 2. 滚动查询：Scroll Search <a href="#usercontent42-gun-dong-cha-xun-scrollsearch" id="usercontent42-gun-dong-cha-xun-scrollsearch"></a>

官方`v7.0+`已不推荐使用滚动查询进行深度分页查询，因为无法保存索引状态。

适合场景：单个[滚动搜索](https://www.elastic.co/guide/en/elasticsearch/reference/7.13/paginate-search-results.html#scroll-search-results)请求中检索大量结果，即非“C端业务”场景

```python
GET <index>/_search?scroll=1m
{
  "size": 100
}

# 上述请求的结果包含一个_scroll_id，应将其传递给scrollAPI 以检索下一批结果。
```

* Scroll上下文的存活时间是滚动的，下次执行查询会刷新，也就是说，不需要足够长来处理所有数据，它只需要足够长来处理前一批结果。保持旧段处于活动状态意味着需要更多的磁盘空间和文件句柄。确保您已将节点配置为具有充足的空闲文件句柄。
* 为防止因打开过多Scrolls而导致的问题，不允许用户打开超过一定限制的Scrolls。默认情况下，打开Scrolls的最大数量为 500。此限制可以通过`search.max_open_scroll_context`集群设置进行更新 。

清除滚动上下文

`scroll`超过超时后，搜索上下文会自动删除。然而，保持Scrolls打开是有代价的，因此一旦不再使用该`clear-scroll`API ，就应明确清除Scroll上下文

```python
# 清除单个
DELETE /_search/scroll
{
  "scroll_id" : "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAD4WYm9laVYtZndUQlNsdDcwakFMNjU1QQ=="
}

# 清除多个
DELETE /_search/scroll
{
  "scroll_id" : [
    "scroll_id1",
    "scroll_id2"
  ]
}

# 清除所有
DELETE /_search/scroll/_all
```

#### 3. Search After

* 不支持向前搜索
* 每次只能向后搜索1页数据
* 适用于C端业务
