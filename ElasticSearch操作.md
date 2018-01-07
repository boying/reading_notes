---
title: ElasticSearch 操作
tags: elasticsearch
grammar_cjkRuby: true
---


## 索引index操作
### 查看所有索引
GET `/_cat/indices?v`

GET `/_alias`

### 查看索引设置
GET `/{index_name}/_setting`

### 创建索引
PUT `/{index_name}`
```
{
    "settings" : {
        "index" : {
            "number_of_shards" : 3, 
            "number_of_replicas" : 2 
        }
    }
}
```
### 创建索引及type
PUT `/{index_name}`
```
{
  "mappings": {
    "tweet" : {
      "properties" : {
        "tweet" : {
          "type" :    "string",
          "analyzer": "english"
        },
        "date" : {
          "type" :   "date"
        },
        "name" : {
          "type" :   "string"
        },
        "user_id" : {
          "type" :   "long"
        }
      }
    }
  }
}
```

### 删除索引
DELETE `/{index_name}`
## type操作
### 创建type
POST `twitter/_mapping/user`
```
{
  "properties": {
    "name": {
      "type": "string"
    }
  }
}
```
```
{
  "properties": {
    "tweet": {
      "type": "string"
    },
    "user": {
      "type": "object",
      "properties": {
        "id": {
          "type": "string"
        },
        "gender": {
          "type": "string"
        },
        "age": {
          "type": "long"
        },
        "name": {
          "type": "object",
          "properties": {
            "full": {
              "type": "string"
            },
            "first": {
              "type": "string"
            },
            "last": {
              "type": "string"
            }
          }
        }
      }
    }
  }
}
```
### 修改type 添加字段
PUT `/gb/_mapping/tweet`
```
{
  "properties" : {
    "tag" : {
      "type" :    "string",
      "index":    "not_analyzed"
    }
  }
}
```
### 删除type OR 删除字段
目前不支持这两个操作

### 查看type的mapping
GET `/jtest/_mapping/employee`

### 查看index下所有type的mapping
GET `/jtest/_mapping/`

## 自定义分析器
### 自定义分析器
PUT `/my_index`
```
{
    "settings": {
        "analysis": {
            "char_filter": {
                "&_to_and": {
                    "type":       "mapping",
                    "mappings": [ "&=> and "]
            }},
            "filter": {
                "my_stopwords": {
                    "type":       "stop",
                    "stopwords": [ "the", "a" ]
            }},
            "analyzer": {
                "my_analyzer": {
                    "type":         "custom",
                    "char_filter":  [ "html_strip", "&_to_and" ],
                    "tokenizer":    "standard",
                    "filter":       [ "lowercase", "my_stopwords" ]
            }}
}}}
```

### 测试分析器
POST `/my_index/_analyze?analyzer=my_analyzer`
```
The quick & brown fox
```


## 文档document操作
### 创建操作
#### 创建document
POST /jtest/employee/1
```
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
```
#### 创建文档，如果id已存在，则不创建
PUT /website/blog/1234/_create
```
{
  "title": "My first blog entry",
  "text": "I am starting to get the hang of this...",
  "date": "2014/01/02"
}
```

### 查找操作
#### 根据id查找
GET `/jtest/employee/1`

#### 查找前10个
GET `/jtest/employee/_search`

#### 根据字段查找
GET `/jtest/employee/_search?q=last_name:Smith`

#### 根据所有字段查找
GET `/jtest/employee/_search?q=Smith`

#### 计数
GET `/jtest/employee/_search?search_type=count`

#### term查找
The term query finds documents that contain the exact term specified in the inverted index.
关注String field是否被analyzed
When querying full text fields, use the match query instead, which understands how the field has been analyzed.
https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html

#### match查找
如果你使用 match 查询一个全文本字段，它会在真正查询之前用分析器先分析match一下查询字符
如果用match下指定了一个确切值，在遇到数字，日期，布尔值或者not_analyzed 的字符串时，它将为你搜索你给定的值

### DSL查询
POST `/jtest/employee/_search`
```
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```
### 复杂的搜索
POST `/jtest/employee/_search`
```
{
    "query" : {
        "filtered" : {
            "filter" : {
                "range" : {
                    "age" : { "gt" : 30 } <1>
                }
            },
            "query" : {
                "match" : {
                    "last_name" : "smith" <2>
                }
            }
        }
    }
}
```
<1> 这部分查询属于区间过滤器(range filter),它用于查找所有年龄大于30岁的数据——gt为"greater than"的缩写。
<2> 这部分查询与之前的match语句(query)一致。

### 全文搜索
POST `/jtest/employee/_search`
```
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
```
默认情况下，Elasticsearch根据结果相关性评分来对结果集进行排序，所谓的「结果相关性评分」就是文档与查询条件的匹配程度。

### 短语搜索（精确全文搜索）
POST `/jtest/employee/_search`
```
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
```
### 聚合 group by
GET `/jtest/employee/_search`
```
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests" }
    }
  }
}
```
```
{
  "query": {
    "match": {
      "last_name": "smith"
    }
  },
  "aggs": {
    "all_interests": {
      "terms": {
        "field": "interests"
      }
    }
  }
}
```
```
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" },
            "aggs" : {
                "avg_age" : {
                    "avg" : { "field" : "age" }
                }
            }
        }
    }
}
```
### 聚合下的聚合，分级汇总
```
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" },
            "aggs" : {
                "avg_age" : {
                    "avg" : { "field" : "age" }
                }
            }
        }
    }
}
```
### 分页
GET `/_search?size=5`
GET `/_search?size=5&from=5`
GET `/_search?size=5&from=10`

（应该当心分页太深或者一次请求太多的结果。结果在返回前会被排序。但是记住一个搜索请求常常涉及多个分片。每个分片生成自己排好序的结果，它们接着需要集中起来排序以确保整体排序正确。）

### 检索多个文档
POST /_mget
```
{
   "docs" : [
      {
         "_index" : "website",
         "_type" :  "blog",
         "_id" :    2
      },
      {
         "_index" : "website",
         "_type" :  "pageviews",
         "_id" :    1,
         "_source": "views"
      }
   ]
}
```

### 检索文档的一部分
GET `/website/blog/123?_source=title,text`

###仅获得_source字段，不要其他元数据
GET `/website/blog/123/_source`

###排序
GET `/jtest/employee/_search`
```
{
  "query": {
    "match_phrase": {
      "about": "I"
    }
  },
  "sort": [
    {
      "age": {
        "order": "desc"
      }
    }
  ]
}
```

### 检查文档是否存在
curl -i -XHEAD http://localhost:9200/website/blog/123

## 更新文档
### 更新整个文档
PUT `/website/blog/123`
```
{
  "title": "My first blog entry",
  "text":  "I am starting to get the hang of this...",
  "date":  "2014/01/02"
}
```
### 跟新文档field
POST `/website/blog/123/_update`
```
{
  "doc": {
    "text": "haha, I am starting to get the hang of this...",
    "date": "2014/01/02"
  }
}
```
### 删除文档
DELETE `/website/blog/123`

## 批量操作
POST `/_bulk`
```
{ "delete": { "_index": "website", "_type": "blog", "_id": "123" }}
{ "create": { "_index": "website", "_type": "blog", "_id": "123" }}
{ "title":    "My first blog post" }
{ "index":  { "_index": "website", "_type": "blog" }}
{ "title":    "My second blog post" }
{ "update": { "_index": "website", "_type": "blog", "_id": "123", "_retry_on_conflict" : 3} }
{ "doc" : {"title" : "My updated blog post"} }


```
注意，最后一行文字后面，要加一个换行符

# 基础概念
## 路由文档到分片
shard = hash(routing) % number_of_primary_shards
routing默认是_id，也可自定义
分片的数量只能在创建索引时定义且不能修改

## 写操作过程
https://raw.githubusercontent.com/looly/elasticsearch-definitive-guide-cn/master/images/elas_0402.png
1. 客户端给Node 1发送写请求
2. 节点使用文档的_id确定文档属于分片0。他转发请求到Node，分片0位于这个节点上
3. Node3在主分片上执行请求，如果成功，它转发请求到相应的位于Node1和Node2的复制节点上。当所有的复制节点报告成功，Node3报告成功到请求节点，请求的节点再报告给客户端

## 检索文档过程
文档能够从主分片或任意一个复制分片被检索
https://raw.githubusercontent.com/looly/elasticsearch-definitive-guide-cn/master/images/elas_0403.png
1. 客户端给Node1发送get请求
2. 节点使用文档的_id确定文档属于的分片0。分片0对应的复制分片在三个节点上都有。此时，它转发到请求到Node2。
3. Node2返回endangered（zhiwen？）给Node1然后返回给客户端。
对于读请求，为了负载均衡，请求节点会为每个请求选择不同的分片——他会循环所有的分片副本。
可能的情况是，一个被索引的文档已经存在于主片上却还没有来得及同步到复制分片上。这是复制分片会报告文档未找到，主分片会成功返回文档。一档索引请求成功返回给用户，文档则在主分片和复制分片都是可用的。

## 局部更新文档
1. 客户端给Node 1发送更新请求。
2. 它转发请求到主分片所在节点Node 3。
3. Node 3从主分片检索出文档，修改_source字段的JSON，然后在主分片上重建索引。如果有其他进程修改了文档，它以retry_on_conflict设置的次数重复步骤3，都未成功则放弃。
4. 如果Node 3成功更新文档，它同时转发文档的新版本到Node 1和Node 2上的复制节点以重建索引。当所有复制节点报告成功，Node 3返回成功给请求节点，然后返回给客户端。

当主分片转发更改给复制分片时，并不是转发更新请求，而是转发整个文档的新版本。记住这些修改转发到复制节点是异步的，它们并不能保证到达的顺序与发送相同。如果Elasticsearch转发的仅仅是修改请求，修改的顺序可能是错误的，那得到的就是个损坏的文档。

## 分析和分析器
#### 分析过程：
1. 标记化一个文本块为适用于倒排索引单独的词（term）
2. 标准化这些词为标准形式，提高它们的“可搜索性”或“查全率”
#### 分析器的功能：
1. 字符过滤器
	字符串先经过字符过滤器，处理文字，去除HTML标记，或者转换为“&”为“and”
2. 分词器
	标记化为单独的词。一个简单的分词器可以根据空格或逗号将单词分开
3. 标记过滤器
	修改词（将“Quick”转为小写），去掉词（“a”、“and”、“the”），或增加词（同义词像“jump”和“leap”）
##查询与过滤
一条过滤语句会询问每个文档的字段值是否包含着特定值；查询语句会询问每个文档的字段值与特定值的匹配程度如何。
条查询语句会计算每个文档与查询语句的相关性，会给出一个相关性评分 _score，并且 按照相关性对匹配到的文档进行排序。 这种评分方式非常适用于一个没有完全配置结果的全文本搜索。
原则上来说，使用查询语句做全文本搜索或其他需要进行相关性评分的时候，剩下的全部用过滤语句
