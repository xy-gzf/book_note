---
description: 索引
---

# Index

## 倒排索引

### 结构

{% tabs %}
{% tab title="term index" %}
词典索引。

词项检索原理：

* Finit State Transducers压缩 fst
{% endtab %}

{% tab title="term dictionary" %}
词项字典。存放词项
{% endtab %}

{% tab title="posting list" %}
倒排表。存储了匹配相对应term的所有id

* int有序数组

两种压缩算法

* roaring bitmaps
* frame of reference
{% endtab %}
{% endtabs %}

#### 倒排索引数据

原始数据

| id | content | other（其他字段） |
| -- | ------- | ----------- |
| 1  | 搞笑视频    |             |
| 2  | 搞笑图片    |             |
| 3  | 搞笑囧图    |             |
| 4  | 伤感视频    |             |
| 5  | 搞笑段子    |             |

倒排数据

| term index | term dictionary | posting list  |
| ---------- | --------------- | ------------- |
|            | 搞笑              | \[1, 2, 3, 5] |
|            | 视频              | \[1, 4]       |
|            | 图片              | \[2]          |
|            | 囧图              | \[3]          |
|            | 伤感              | \[4]          |
|            | 段子              | \[5]          |

### posting list 压缩算法

#### **frame of reference**

通过将数组进行错位相减，获得一个每个值更小的数组

\[1, 2, 3, 5]  >> \[1, 1, 1, 2]

节省每个数值存储空间，然后通过动态分组，压缩后存储下来

* 适用于差值较小的数组

#### **roaring bitmaps**

将数组每个数字对65536取除数及余数

\[1, 2, 3, 5]  >>  \[(0, 1), (0, 2), (0, 3), (0, 5)]

除数通过一个short数组存储，相同数只存一个。每个值作为一个key

每个key通过一个container存储余数。该container可以是array、bitmap及runcontainer

key: \[0]

arraycontainer: \[\[1, 2, 3, 5]]

bitmapcontainer: \[46(0... 0010 1110)] &#x20;

runcontainer:&#x20;

### term dictionary压缩

ES通过**有限状态转换器**，把term dictionary变成了term index，放进了内存，所以这个term index到底长什么样呢？

#### **FST 有限状态转换器（Finite State Transducers）**

相当于是一个Trie前缀树，可以直接根据前缀就找到对应的term在词典中的位置。

比如我们现在有已经排序好的单词mop、moth、pop、star、stop和top以及他们的顺序编号0..5，可以生成一个下面的状态转换图

<figure><img src="https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FxueVvbeE0N0K1eGhMGM7%2Fuploads%2F6pTO2uxW6M985rLJzu5O%2Ffst.png?alt=media&#x26;token=1a00863b-3c66-4b2b-ab82-eb385586b16c" alt=""><figcaption><p>fst</p></figcaption></figure>

