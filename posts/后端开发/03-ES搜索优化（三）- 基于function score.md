---
title: ES搜索优化（三）- 基于function score
description: ES搜索优化（三）- 基于function score
date: 2023-03-03
tags:
  - 后端开发
  - Elasticsearch
---

# 1. 什么是function score？
+ function score 就是 elasticsearch 提供的一种通过**函数**来对相关性评分进行二次计算的方法。这里的函数可以大致分为两种。
    - 第一种：script_score 我们开发人员自己通过 plain  painless 进行编写的。
    - 第二种：elasticsearch 提供的。
        * weight : 加权。
        * random_score : 随机打分。
        * field_value_factor : 使用字段的数值参与计算分数。
        * decay_function : 衰减函数 gauss, linear, exp 等。

# 2. 为什么需要function score？
+ 我们做搜索出来的数据排序的时候，往往不能只考虑字符的相关性，还需要考虑产品方的各种定制化的需求。而function score则可以用来解决定制化需求。
+ 这里有可能有同学会说，我们可以把数据搜索出来以后，自己用自己熟悉的编程语言进行分数计算，以及排序。这是一种办法，但是会有两个问题。
    - 第一个：我们需要把所有数据先查询出来，然后再一起进行计算，这里全部查出来这个动作对于资源的消耗会非常大。
    - 第二个：我理解elasticsearch提供的函数，支持组合使用的情况下，是可以满足大部分业务需求的，我们就可以不用再自己造轮子了。

# 3. function对应解决的业务需求？
## 3.1 script score（解决复杂的自定义需求）
+ **可以解决的业务需求：**可太多了，因为支持自己写一个函数去针对score进行计算，函数中可以获取到每个document的值，以及这个document当下的分数。
+ **代码示例（一）**：相关性分数等于某个值/10，同时由于这是一个函数脚本，其实还可以写更多的逻辑，可以看第二个示例

```shell
GET /_search
{
  "query": {
    "script_score": {
      "query": {
        "match": { "message": "elasticsearch" }
      },
      "script": {
        "source": "doc['my-int'].value / 10 "
      }
    }
  }
}
```

+ **代码示例（二）**：
    -  这里可以看到通过函数实现余弦相似度，当然这里强调的重点不是余弦相似度，而是我们可以用这个实现很复杂的计算逻辑。从而对于相关性分数有一个兜底计算方案。

```go
GET my-index-000001/_search
{
  "query": {
    "script_score": {
      "query" : {
        "bool" : {
          "filter" : {
            "term" : {
              "status" : "published"
            }
          }
        }
      },
      "script": {
        "source": """
          float[] v = doc['my_dense_vector'].vectorValue;
          float vm = doc['my_dense_vector'].magnitude;
          float dotProduct = 0;
          for (int i = 0; i < v.length; i++) {
            dotProduct += v[i] * params.queryVector[i];
          }
          return dotProduct / (vm * (float) params.queryVectorMag);
        """,
        "params": {
          "queryVector": [4, 3.4, -0.2],
          "queryVectorMag": 5.25357
        }
      }
    }
  }
}
```

## 3.2 weight
+ **可以解决的业务需求**：通过对文档进行加权。解决比如我们搜索文章，那么文章title命中关键词得到的分数，应该比文章content命中关键词得到的分数更高。
    - weight 是 5 ，即自定义函数得分 func_score = 5 ，最终结果的 score 等于 query_score * 5

```shell
{
  "query": {
    "function_score": {
      "query": { "match": { "message": "elasticsearch" } },
      "functions": [
        {
          "filter": { "match": { "title": "elasticsearch" } },
          "weight": 5
        }
      ]
    }
  }
}
```

## 3.3 random
+ **可以解决的业务需求**：通过随机打分，生成 [0, 1) 之间均匀分布的随机分数值。可以解决比如我们的广告推荐，我们希望每个用户都能看到一个不同的随机顺序（尽可能多的曝光不同的广告），但是对于相同的用户，当他点击第二页，第三页或者后续页面时，看到的顺序应该是相同的。这就是所谓的一致性随机(Consistently Random)
+ **示例**：
    - random_score 影响因素之一（seed），比如用户ID，这样每一个用户看到的数据都是不一样的。
    - random_score 影响因素之二（field），可以理解为是另外一个 seed。当某个字段发生变化的时候，整 最终生成的 random score 也会发生变化。

```shell
GET test_index/_search
{
  "query": {
    "function_score": {
      "random_score": {
        "seed": 10,
        "field": "id"
      }
    }
  }
}
```

## 3.4 field value factor
+ **可以解决的业务需求**：把一个文章的 popularity 或 votes （受欢迎或赞）作为相关性分数计算中的一部分，比如同样命中关键词的文章，点赞更多的应该更加前面( 哪怕命中的关键词稍微少一点 )。
+ 举例：

```shell
{
  "query": {
    "function_score": {
      "query": { "match": { "message": "elasticsearch" } },
      "field_value_factor": {
        "field": "likes",
        "factor": 1.2,
        "missing": 1,
        "modifier": "log1p"
      }
    }
  }
}
```

+ field : 参与计算的字段。
+ factor : 乘积因子，默认为 1 ，将会与 field 的字段值相乘。
+ missing : 如果 field 字段不存在则使用 missing 指定的缺省值。
+ modifier : 计算函数，为了避免分数相差过大，用于平滑分数，可以是以下之一：
    - none : 不处理，默认
    - log : log(factor * field_value)
    - log1p : log(1 + factor * field_value)
    - log2p : log(2 + factor * field_value)
    - ln : ln(factor * field_value)
    - ln1p : ln(1 + factor * field_value)
    - ln2p : ln(2 + factor * field_value)
    - square : 平方，(factor * field_value)^2
    - sqrt : 开方，sqrt(factor * field_value)
    - reciprocal : 求倒数，1/(factor * field_value)
+ 假设某个匹配的文档的点赞数是 1000 ，那么例子中其打分函数生成的分数就是 log(1 + 1.2 * 1000)，最终的分数是原来的 query 分数与此打分函数分数相差的结果。

## 3.5 decay function
+ **可以解决的业务需求**：通过一个函数，对某个数值进行一个递减的操作，然后将递减后的数值融入到相关性分数计算中。从而解决:比如我们搜索外卖的时候，肯定是地理位置离我们更近的店要展示的在更加前面。又比如我们索一些研究论文的时候，发布时间更新的，肯定也是需要更加靠前。
+ **代码示例（一）**：

```shell
"DECAY_FUNCTION": {
    "FIELD_NAME": {
          "origin": "30, 120",
          "scale": "2km",
          "offset": "0km",
          "decay": 0.33
    }
}
```

上例的意思就是在距中心点方圆 2 公里之外，分数减少到三分之一（乘以 decay 的值 0.33）。

+ DECAY_FUNCTION 可以是以下任意一种函数：
    - linear : 线性函数
    - exp : 指数函数
    - gauss : 高斯函数
+ origin : 中心点，只能是数值、日期、geo-point
+ scale : 定义到中心点的距离
+ offset : 偏移量，默认 0
+ decay : 衰减指数，默认是 0.5



+ **代码示例（二）**

```shell
GET /_search
{
  "query": {
    "function_score": {
      "gauss": {
        "@timestamp": {
          "origin": "2013-09-17",
          "scale": "10d",
          "offset": "5d",
          "decay": 0.5
        }
      }
    }
  }
}
```

+ 中心点是 2013-09-17 日期，scale 是 10d 意味着日期范围是 2013-09-12 到 2013-09-22 的文档分数权重是 1 ，日期在 scale + offset = 15d 之外的文档权重是 0.5 。



# 4. 调试相关性时的一些监控指标
+ **可以解决的业务需求**：我们对搜索进行优化，是希望让客户能够更快的从数据库是检索到他想要的数据，而怎么判断我们的优化是否真的起到了作用，我们就可以通过一些标准来进行判断。我这里提供两种方式。
    - 第一种：搜索引擎常用的 [DCG](https://baike.baidu.com/item/DCG)：
        * 搜索引擎一般采用PI（per item）的方式进行评测，简单地说就是逐条对搜索结果进行分等级的打分。假设我们在Google上搜索一个词，然后得到5个结果。我们对这些结果进行3个等级的区分：Good（好）、Fair（一般）、Bad（差），然后赋予他们分值分别为3、2、1，假定通过逐条打分后，得到这5个结果的分值分别为3、2 、1 、3、 2。然后再对这些分数进行一些数学计算，具体可以参看：[https://baike.baidu.com/item/DCG](https://baike.baidu.com/item/DCG)
    - 第二种：简单易用的的数据埋点，结合个人的业务经验。
        * 用户点击最顶端结果的频次，这可以是前 10 个文档，也可以是第一页的；
        * 用户不查看首次搜索的结果而直接执行第二次查询的频次；
        * 用户来回点击并查看搜索结果的频次；
        * 等等诸如此类的信息。

# 5. 参考
+ [https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-function-score-query.html](https://www.elastic.co/guide/en/elasticsearch/reference/7.1/query-dsl-function-score-query.html)
+ [https://www.liujiajia.me/2019/10/21/elasticsearch-score](https://www.liujiajia.me/2019/10/21/elasticsearch-score)
+ [https://segmentfault.com/a/1190000037700644](https://segmentfault.com/a/1190000037700644)

