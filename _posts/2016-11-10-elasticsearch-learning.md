---
layout: post
date: 2016-10-25
title: ElasticSearch Learning Part 1
categories: [elasticsearch]
---

## Elasticsearch Search

### Structured Search
结构化数据搜索，搜索有结构的数据，比如date，time，numbers，有固定的格式，可以有逻辑运算，比如比较等。

有些情况下，text也可以用结构化搜索，比如一个field可以有一些固定的值，请参考ref。

结构化搜索的结果只能是yes or no，或者属于set 或者不属于，结构化搜索不关注doc关联和scoring

ref：https://www.elastic.co/guide/en/elasticsearch/guide/current/structured-search.html

#### 查找确切的值
这种情况下，搜索只需和no-scoring，filter配合使用。Filter是非常有用的工具，他会加快查询速度。这不需要计算关联（关联会在做scoring的时候做），并且能非常方便的cache。

##### term query with number
term能够处理number，bool，date，text等。是非常常用的查询方式。

term用于匹配完全一致的值。

通常，在查找完全一致的值的时候，我们并不需要做scoring。我们只是想要查找，包含或者不包含的情况。这样我们需要用`constant_score`来执行`term`

~~~json
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "term" : {
                    "price" : 20
                }
            }
        }
    }
}
~~~

##### term query with text
当我们在用term来查询text的时候，要注意对于text域，默认是会有`analyze`操作的，这个时候查的term并不会做`analyze`后去查。具体细节请看ref

如果我们要对某个域做term查找，要把这个域设置成`not_analyzed`

~~~json
DELETE /my_store

PUT /my_store
{
    "mappings" : {
        "products" : {
            "properties" : {
                "productID" : {
                    "type" : "string",
                    "index" : "not_analyzed"
                }
            }
        }
    }

}
~~~

##### Internal Filter Operation
当执行`non-scoring`的时候，ES会在内部执行很多操作.

1. Find matching docs
`term` 会去倒排索引中查询并返回所有的doc

2. Build a bitset
Filter会对所有doc都计算一个bitset，标记哪些doc有这个term

3. 遍历bitset
当所有querey都执行完之后，遍历所有的bitset，找到满足全部query的doc。bitset的顺序很重要，一般会先遍历稀疏的bitset

4. Increment the usage count
ES可以cache non-scoring queries, 但是cache不常用的query非常愚蠢，我们其实可以只cache那些近期使用频繁的query。为了做到这一点，ES会保存query的历史，如果一个query在最近256次查询中，查询多余了一定的值，我们就把这个query cache起来。当segments个数少于10k或者不超过index size的3%时，将不会cache，因为这些segments耗费不大，但是对cache确是浪费。

#### Combining Filters
如何构造一个复杂的filter逻辑
~~~sql
SELECT product
FROM   products
WHERE  (price = 20 OR productID = "XHDK-A-1293-#fJ3")
  AND  (price != 30)
~~~

这需要在`constant_score`里面用到`bool` query

##### Bool Filter
~~~json
{
   "bool" : {
      "must" :     [],
      "should" :   [],
      "must_not" : [],
      "filter":    []
   }
}
~~~

Bool Filter 包含4种section。

- `must`：等同于`AND`
- `must_not`: `NOT`
- `should`: `OR`
- `filter`: 等同于`must`, 但是是non-scoring，filter模式

由于我们现在讨论的是在`constant_score`下，已经是non-scoring 模式了，所以不讨论`filter`。

上面sql对应的query为

~~~json
GET /my_store/products/_search
{
   "query" : {
      "constant_score" : {
         "filter" : {
            "bool" : {
              "should" : [
                 { "term" : {"price" : 20}},
                 { "term" : {"productID" : "XHDK-A-1293-#fJ3"}}
              ],
              "must_not" : {
                 "term" : {"price" : 30}
              }
           }
         }
      }
   }
}
~~~

##### Nesting Boolean Queries
更复杂一点，query 条件为
~~~ sql
SELECT document
FROM   products
WHERE  productID      = "KDKE-B-9947-#kL5"
  OR (     productID = "JODL-X-1937-#pV7"
       AND price     = 30 )
~~~

~~~json
GET /my_store/products/_search
{
   "query" : {
      "constant_score" : {
         "filter" : {
            "bool" : {
              "should" : [
                { "term" : {"productID" : "KDKE-B-9947-#kL5"}},
                { "bool" : {
                  "must" : [
                    { "term" : {"productID" : "JODL-X-1937-#pV7"}},
                    { "term" : {"price" : 30}}
                  ]
                }}
              ]
           }
         }
      }
   }
}
~~~

#### Finding Multiple Exact Value
`term`是用来查找单一值的，但当我们需要查找多个值的时候该怎么办？这个时候我们可以用`terms`来做。

~~~json
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "terms" : {
                    "price" : [20, 30]
                }
            }
        }
    }
}
~~~

##### Contains, but does not equal
`term` 和 `terms` 都是 _contain_ 算子，不是 _equal_

##### Equals Exactly
如果想要完全匹配，一般来说需要做第二个filter，限制这个field的term个数。

~~~json
GET /my_index/my_type/_search
{
    "query": {
        "constant_score" : {
            "filter" : {
                 "bool" : {
                    "must" : [
                        { "term" : { "tags" : "search" } },
                        { "term" : { "tag_count" : 1 } }
                    ]
                }
            }
        }
    }
}
~~~

#### Ranges
- gt: > greater than
- lt: < less than
- gte: >= greater than or equal to
- lte: <= less than or equal to

~~~json
GET /my_store/products/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "range" : {
                    "price" : {
                        "gte" : 20,
                        "lt"  : 40
                    }
                }
            }
        }
    }
}
~~~

range对比string是按字典序。

#### Dealing with Null Values
一个index有一个field叫tags，这个tags可能含有一个tag，多个tag，也可能没有tag。那么没有tag的时候是怎么存倒排索引的呢？

其实并没有存。`null`, `[]`, `[null]`是全部一样的，他们都不存在于倒排索引中。

##### Exists query
使用方法

~~~json
GET /my_index/posts/_search
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "exists" : { "field" : "tags" }
            }
        }
    }
}
~~~

##### missing query
使用方法

~~~json
GET /my_index/posts/_search
{
    "query" : {
        "constant_score" : {
            "filter": {
                "missing" : { "field" : "tags" }
            }
        }
    }
}
~~~

#### All about caching
cache是增量更新的，比如一个新的document插入了，那么被cache的query会把新的document计算并添加到cache中。

##### Independent Query Cache
Cache是独立于query之外的，一个query的多个bitset是独立cache的。
