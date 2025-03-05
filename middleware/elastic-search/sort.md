---
description: 排序
---

# Sort

## Basics Score

### 评分

`es5` 开始，默认的相似度算法是BM25，BM25有两个可调参数k1 及 b

* `es5`之前使用tf-idf算法（es5还可以使用tf-idf算法）
* `es7`直接删除了tf-idf算法

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FwHj0mwaVGQsRdbCAuIyN%2Fuploads%2Fb2rQF3F2ahZF4GK9d9ep%2Fbm25.png?alt=media&#x26;token=003541ff-1f44-4b05-8f17-9feaacbd6eab" alt=""><figcaption><p>bm25</p></figcaption></figure>

*   **k1**

    * 用于确定项频率饱和度特性的变量。限制单个查询项对给定文档分数的影响程度

    <figure><img src="https://1301551370.vod-qcloud.com/dc9dab3cvodcq1301551370/1b36e12a1253642696719360681/aPrLbswtdZoA.png" alt="" width="563"><figcaption><p>tf/idf对比bm25</p></figcaption></figure>
* **b**
  * b越大，文档长度与平均长度之比的影响就越大。b设为0，长度比则会失效（如果不想因为长度削弱关联性，b的值设置低一点比较合适）

{% embed url="https://www.elastic.co/cn/blog/practical-bm25-part-3-considerations-for-picking-b-and-k1-in-elasticsearch?ref=farer.org" %}
如何选取k1和b的参数
{% endembed %}

#### TF\_term frequency

**关键词在每个doc中出现的次数**

`freq / (freq + k1 * (1 - b + b * dl / avgdl))`

```
{
    'freq':  x,     # 词在当前文档中出现的次数
    'k1':    1.2,   # 可调参数  默认1.2（0~3 最优：0.5～2.0）
    'b':     0.75,  # 可调参数  默认0.75（0～1 最优：0.3～0.9）
    'dl':    x,     # 本文档分词后分词单元个数
    'avgdl': x,     # 平均dl，所有文档分词后分词单元总和 / 文档数
}
```

#### IDF\_inverse doc frequency

**关键词在整个索引中出现的次数**

`log(1 + (N - n + 0.5) / (n + 0.5))`

```
{
    'N': x,     # 待检索文档数
    'n': x,     # 包含分词的文档数目
}
```

#### score总评分

```
# 如果不指定boost，boost 就是使用缺省值，缺省值是 2.2
# 总评分 = 搜索词分词后每个词的boost * TF * IDF 相加
score = (boost * TF * IDF) + (boost * TF * IDF) + ...
```

## Function Score

#### 查询方式

```
GET /_search
{
  "query": {
    "function_score": {
      "query": { "match_all": {} },
      "boost": "5",
      "random_score": {}, 
      "boost_mode": "multiply"
    }
  }
}
```

#### 支持的函数 <a href="#zhi-chi-de-han-shu" id="zhi-chi-de-han-shu"></a>

* ​[`script_score`](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/query-dsl-function-score-query.html#function-script-score)​Comment
* ​[`weight`](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/query-dsl-function-score-query.html#function-weight)​Comment
* ​[`random_score`](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/query-dsl-function-score-query.html#function-random)​Comment
* ​[`field_value_factor`](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/query-dsl-function-score-query.html#function-field-value-factor)​Comment
* ​[decay functions](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/query-dsl-function-score-query.html#function-decay): `gauss`, `linear`, `exp`Comment

#### score\_mode支持参数 <a href="#scoremode-zhi-chi-can-shu" id="scoremode-zhi-chi-can-shu"></a>

| `multiply` | scores are multiplied (default)                          |
| ---------- | -------------------------------------------------------- |
| `sum`      | scores are summed                                        |
| `avg`      | scores are averaged                                      |
| `first`    | the first function that has a matching filter is applied |
| `max`      | maximum score is used                                    |
| `min`      | minimum score is used                                    |

#### boost\_mode支持参数 <a href="#boostmode-zhi-chi-can-shu" id="boostmode-zhi-chi-can-shu"></a>

| `multiply` | query score and function score is multiplied (default)  |
| ---------- | ------------------------------------------------------- |
| `replace`  | only function score is used, the query score is ignored |
| `sum`      | query score and function score are added                |
| `avg`      | average                                                 |
| `max`      | max of query score and function score                   |
| `min`      | min of query score and function score                   |
