# ES 基础查询
{docsify-updated}

## RestFul API
`curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'`

VERB-适当的 HTTP 方法: GET、 POST、 PUT、 HEAD 或者 DELETE。

### 集群健康
```
curl -XGET 'localhost:9200/_cat/health?v&pretty'
GET /_cat/health
```

节点列表:
```
curl -XGET 'localhost:9200/_cat/nodes?v&pretty'
GET /_cat/nodes
```

### 索引增删查（索引创建好后不允许修改）
列出所有索引:
```
curl -XGET 'localhost:9200/_cat/indices?v&pretty'
GET /_cat/indices
```

创建索引，索引名必须小写，不能以下划线开头，不能包含逗号:
```
curl -XPUT 'localhost:9200/customer?pretty&pretty'
PUT /customer


PUT fund
{
  "settings": {
    "index": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "analysis": {
        "tokenizer": {
          "chinese_chars_tokenizer": {
            "type": "pinyin",
            "keep_first_letter": false,
            "keep_separate_first_letter": false,
            "keep_full_pinyin": false,
            "keep_original": false,
            "limit_first_letter_length": 50,
            "keep_separate_chinese": true,
            "lowercase": true
          },
          "pinyin_tokenizer": {
            "type" : "pinyin",
            "keep_first_letter":true,
            "keep_separate_first_letter" : true,
            "keep_full_pinyin" : true,
            "keep_original" : false,
            "limit_first_letter_length" : 16,
            "lowercase" : true
          }
        },
        "char_filter": {
          "remove_spaces": {
            "type": "pattern_replace",
            "pattern": """\s+""",
            "replacement": ""
          }
        },
        "analyzer": {
          "pinyin_analyzer": {
            "tokenizer": "pinyin_tokenizer"
          },
          "chinese_pinyin_search_analyzer" : {
            "tokenizer": "chinese_chars_tokenizer"
          },
          "ngram_lowercase_analyzer": {
            "tokenizer": "ngram",
            "filter": [
              "lowercase"
            ],
            "char_filter": [
              "remove_spaces"
            ]
          }
        }
      }
    }
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "isin": {
        "type": "text",
        "analyzer": "ngram_lowercase_analyzer",
        "search_analyzer": "ngram_lowercase_analyzer",
        "fields": {
          "keyword":{
            "type": "keyword"
          }
        }
      },
      "nameEnUs": {
        "type": "text",
        "analyzer": "ngram_lowercase_analyzer",
        "search_analyzer": "ngram_lowercase_analyzer",
        "fields": {
          "keyword":{
            "type": "keyword"
          }
        }
      },
      "nameZhCn": {
        "type": "text",
        "analyzer": "ngram_lowercase_analyzer",
        "search_analyzer": "ngram_lowercase_analyzer",
        "fields": {
          "pinyin": {
            "type": "text",
            "analyzer": "pinyin_analyzer",
            "search_analyzer": "chinese_pinyin_search_analyzer"
          },
          "keyword":{
            "type":"keyword"
          }
        }
      },
      "nameZhHk": {
        "type": "text",
        "analyzer": "ngram_lowercase_analyzer",
        "search_analyzer": "ngram_lowercase_analyzer",
        "fields": {
          "pinyin": {
            "type": "text",
            "analyzer": "pinyin_analyzer",
            "search_analyzer": "chinese_pinyin_search_analyzer"
          },
          "keyword":{
            "type":"keyword"
          }
        }
      },
      "companyEnUs": {
        "type": "text",
        "analyzer": "ngram_lowercase_analyzer",
        "search_analyzer": "ngram_lowercase_analyzer",
        "fields": {
          "keyword":{
            "type": "keyword"
          }
        }
      },
      "companyZhCn": {
        "type": "text",
        "analyzer": "ngram_lowercase_analyzer",
        "search_analyzer": "ngram_lowercase_analyzer",
        "fields": {
          "pinyin": {
            "type": "text",
            "analyzer": "pinyin_analyzer",
            "search_analyzer": "standard"
          },
          "keyword":{
            "type":"keyword"
          }
        }
      },
      "companyZhHk": {
        "type": "text",
        "analyzer": "ngram_lowercase_analyzer",
        "search_analyzer": "ngram_lowercase_analyzer",
        "fields": {
          "pinyin": {
            "type": "text",
            "analyzer": "pinyin_analyzer",
            "search_analyzer": "standard"
          },
          "keyword":{
            "type":"keyword"
          }
        }
      },
      "currency": {
        "type": "keyword"
      },
      "isEmployee": {
        "type": "boolean"
      },
      "isCrossSouthPro": {
        "type": "integer"
      },
      "isSj": {
        "type": "boolean"
      },
      "isDel": {
        "type": "boolean"
      },
      "isOnlyProfessionalInvestor": {
        "type": "boolean"
      },
      "channel": {
        "type": "keyword"
      },
      "assetClass": {
        "type": "keyword"
      }
    }
  }
}
```

删除索引：
```
curl -XDELETE 'localhost:9200/customer?pretty&pretty'
DELETE /customer
```

### 文档增删改查
将文档放入customer索引，ID置为1:
```
curl -XPOST 'localhost:9200/customer/1?pretty&pretty' -d'
{
  "name": "John Doe"
}'

POST /customer/1
{
  "name": "John Doe"
}
```

自动生成ID:
```
POST /customer
{
  "name": "John Doe2"
}
```

查询文档：
```
curl -XGET 'localhost:9200/customer/1?pretty&pretty'
GET /customer/_doc/1 //查询指定ID的文档
HEAD hk_market_code_table/_doc/00555-HKG //只检查对应ID的文档是否存在
POST /customer/_search //查询所有文档,默认返回10条
GET hk_market_code_table/_count 查询某个索引下文档总数
```