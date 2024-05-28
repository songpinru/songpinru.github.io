概念：

index：表

type：摆设

document：一条数据

field：字段，表头

shard 分片

node ：节点

后台启动（daemon）

./bin/elasticsearch -d -p ./pid

# 操作

```
Kibana写法：
REST /INDEX/TYPE/<ID>
```

## PUT

INDEX：

```sh
curl -X PUT "localhost:9200/customer?pretty"
# PS：？pretty的意思是响应，以JSON格式返回
```

DOCUMENT:

```sh
curl -X PUT "localhost:9200/customer/_doc/1?pretty" -H 'Content-Type: application/json' -d'{"name": "John Doe"}'
```

PUT数据到index，只能新增或覆盖（整体性），不可以部分更改

## POST

DOCUMENT：

```sh
# 文档修改
curl -X POST "localhost:9200/customer/_doc/1/_update?pretty" -H 'Content-Type: application/json' -d'
{
  "doc": { "name": "Jane Doe", "age": 20 }
}'

# 文档升级
curl -X POST "localhost:9200/customer/_doc/1/_update?pretty" -H 'Content-Type: application/json' -d'
{
  "script" : "ctx._source.age += 5"
}'
# PS：？pretty的意思是响应，以JSON格式返回
# ctx._source引用的是当前源文档
```

## DELETE

INDEX：

```sh
curl -X DELETE "localhost:9200/customer?pretty"
# PS：？pretty的意思是响应，以JSON格式返回
```

DOCUMENT：

```sh
curl -X DELETE "localhost:9200/customer/_doc/2?pretty"
```

## GET

INDEX:

```shell
# health：集群健康
curl -X GET "localhost:9200/_cat/health?v"
# nodes：节点列表
curl -X GET "localhost:9200/_cat/nodes?v"
# indices：全部索引
curl -X GET "localhost:9200/_cat/indices?v"

# PS：末尾的 ?v 表示显示字段名
```

status的三种状态：

- Green ： everything is good（一切都很好）（所有功能正常）
- Yellow ： 所有数据都是可用的，但有些副本还没有分配（没有副本）
- Red ： 有些数据不可用（部分功能正常）

> PS：字段中的**pri**为分片数，rep为副本数
> 	在默认情况下，Elasticsearch中的每个索引都分配了5个主分片和1个副本

DOCUMENT：

```sh
curl -X GET "localhost:9200/customer/_search?pretty
curl -X GET "localhost:9200/customer/_doc/1?pretty
curl -X GET "localhost:9200/customer/_mapping?pretty
# PS：？pretty的意思是响应，以JSON格式返回
```

## 批处理

除了能够索引、更新和删除单个文档之外，Elasticsearch还可以使用**_bulk** API批量执行上述任何操作。

```sh
curl -X POST "localhost:9200/customer/_doc/_bulk?pretty" -H 'Content-Type: application/json' -d'
{"index":{"_id":"1"}}
{"name": "John Doe" }
{"index":{"_id":"2"}}
{"name": "Jane Doe" }
'


curl -X POST "localhost:9200/customer/_doc/_bulk?pretty" -H 'Content-Type: application/json' -d'
{"update":{"_id":"1"}}
{"doc": { "name": "John Doe becomes Jane Doe" } }
{"delete":{"_id":"2"}}
'
```

## mapping

```sh
GET movie_index/_mapping/movie

PUT movie_chn
{
  "mappings": {
    "movie":{
      "properties": {
        "id":{
          "type": "long"
        },
        "name":{
          "type": "text"
          , "analyzer": "ik_smart"
        },
        "doubanScore":{
          "type": "double"
        },
        "actorList":{
          "properties": {
            "id":{
              "type":"long"
            },
            "name":{
              "type":"keyword"
            }
          }
        }
      }
    }
  }
}

```

## aliases

```bash
GET  _cat/aliases?v

PUT movie_chn_2020
{  "aliases": {
      "movie_chn_2020-query": {}
  }, 
  "mappings": {
    "movie":{
      "properties": {
        "id":{
          "type": "long"
        },
        "name":{
          "type": "text"
          , "analyzer": "ik_smart"
        },
        "doubanScore":{
          "type": "double"
        },
        "actorList":{
          "properties": {
            "id":{
              "type":"long"
            },
            "name":{
              "type":"keyword"
            }
          }
        }
      }
    }
  }
}



POST  _aliases
{
    "actions": [
        { "add":    { "index": "movie_chn_xxxx", "alias": "movie_chn_2020-query" }}
    ]
}
POST  _aliases
{
    "actions": [
        { "remove":    { "index": "movie_chn_xxxx", "alias": "movie_chn_2020-query" }}
    ]
}
POST /_aliases
{
    "actions": [
        { "remove": { "index": "movie_chn_xxxx", "alias": "movie_chn_2020-query" }},
        { "add":    { "index": "movie_chn_yyyy", "alias": "movie_chn_2020-query" }}
    ]
}

```

## template

```sh
GET  _cat/templates

GET  _template/template_movie2020
或者
GET  _template/template_movie*

PUT _template/template_movie2020
{
  "index_patterns": ["movie_test*"], #以movie_test开头的都符合               
  "settings": {                                               
    "number_of_shards": 1
  },
  "aliases" : { 
    "{index}-query": {},#索引后面加-query的别名
    "movie_test-query":{}
  },
  "mappings": {                                          
	"_doc": {
      "properties": {
        "id": {
          "type": "keyword"
        },
        "movie_name": {
          "type": "text",
          "analyzer": "ik_smart"
        }
      }
    }
  }
}

```



## 查询

```sh
curl -X GET "localhost:9200/bank/_search" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "sort": { "balance": { "order": "desc" } },
  "_source": ["account_number", "balance"],
  "from": 10,
  "size": 10
}
'
```

* query

  * match_all
  * match_none
  * match

    * 字段名
      * query：字段内容
      * oprator：or/and
      * analyzer："ik_smart"分词器
      * 
  * match_phrase

    * 字段名

      - query：字段内容

      - analyzer：文本解析器

      - max_expansions：末尾单词可扩展后缀数
  * mult_match

    * 
  * fuzzy
  * boosting

    * positive
      * term
      * match
    * negative
      * term
      * match
    * negative_boost(降低系数)
  * constant_score
    * filter
    * boost(默认1.0,相关性得分恒定)
  * dis_max
    * queries
    * tie_breaker(最高匹配度之外的查询得分系数)
  * nested
    * path
    * query
    * score_mode
      * avg
      * max
      * min
      * none
      * sum
    * ignore_unmapped
      * flase
      * true(path找不到的时候不报错)
  * bool
    * must
      * match
      * term
    * should
      * match
      * term
    * must_not
      * match
      * range
    * filter
      * term
      * range
        * 大于：gt
        * 小于：lt
        * 大于等于：gte 
        * 小于等于：lte 
* sort

  * 字段数组
    * order：desc/asc
* aggs

  * NAME:分组名
    * avg
    * sum
    * max
    * min
    * stats
    * terms
      * field：字段名(.keyword)
      * order
        * 字段名：desc/asc
      * avg
        * field
      * aggs
      * stats
      * size:默认为10
* post_filter
    * term
    * range
      - 大于：gt
      - 小于：lt
      - 大于等于：gte 
      - 小于等于：lte 
* from
* size
* _source
* highlight

  * fields



```json
GET /gmall_sale_detail/doc/_search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "date": "2020-04-17"
        }
      },
      "must": [
        {"match": {
          "name": {
            "query": "xiaomi",
            "operator": "and"
          }
        }}
      ]
    }
  },
  "aggs": {
    "groupby_age": {
      "terms": {
        "field": "age",
        "size": 100
      }
    },
    "groupby_gender":{
      "terms": {
        "field": "gender",
        "size": 2
      }
    }
  },
  "from": 0,
  "size": 20,
  "highlight": {
    "fields": {"name":{}}
  }
}
```



<details>
<summary>CLICK </summary>
adddssf
sds
dfsdf
dfs
</details>



```xml
<details>
<summary>CLICK ME</summary>

**<summary>标签与正文间一定要空一行！！！**
</details>
```