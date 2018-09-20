# Elasticsearch

[TOC]

# 一、Elasticsearch入门

## 0、常见术语

- 文档 Document
  - 用户存储在es中的数据文档
  - 可以理解为数据库中表的一行记录
- 索引 Index
  - 由具有相同字段的文档列表组成
  - 在6.0版本以后可以理解为数据库的表，不在适合理解为数据库了，因为在索引下只能创建一个type
- 节点 Node
  - 一个Elasticsearch的运行实例，是集群的构成单元
- 集群
  - 由一个或多个节点组成，对外提供服务

## 1、Document

- Json Object，由字段（Field）组成，常见数据类型如下：
  - 字符串：text，keyword
  - 数值型：byte，short，integer，long，float，double，half_float，scaled_float
  - 布尔：boolean
  - 日期：date
  - 二进制：binary
  - 范围类型：integer_range，long_range，float_range，double_range，date_range
- 每个文档有唯一的id标识
  - 自行指定
  - es自动生成
  - 类似于数据库表的主键
- 元数据，用于标注文档的相关信息
  - _index：文档所在的索引名
  - _type：文档所在的类型名
  - _id：文档唯一id
  - _uid：组合id，由 _type和 _id组成（6.x _type不再起作用，同 _id一样）
  - _source：文档的原始Json数据，可以从这里获取每个字段的内容
  - _all：整合所有字段内容到该字段，默认禁用，不推荐使用

## 2、Index

- 索引中存储具有相同机构的文档（Document）
  - 每个索引都有自己的`mapping`定义，用于定义字段名和类型
- 一个集群可以有多个索引，比如：
  - nginx 日志存储的时候可以按照日期每天生成一个索引来存储
    - nginx-log-2017-01-01
    - nginx-log-2017-01-02
    - nginx-log-2017-01-03

## 3、Rest API

- Elasticsearch集群对外提供RESTful API
  - REST - REpresentational State Transfer
  - URI指定资源，如Index、Document等
  - Http Method指定资源操作类型，如GET、POST、PUT、DELETE等
    - GET 获取资源
    - POST 修改或更新资源
    - PUT 新建资源
    - DELETE 删除资源
- 常用两种交互方式
  - Curl命令行
  - Kibana DevTools

## 4、索引API

### 4.1、创建索引api如下：

```
request:
PUT /test_index
response:
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "test_index"
}
```

### 4.2、查看现有索引

```
GET _cat/indices
```

### 4.3、删除索引

```
request:
DELETE /test_index
response:
{
  "acknowledged": true
}
```

## 5、文档 Document API

### 5.1、创建文档

**创建文档时，如果索引不存在，es会自动创建对应的index和type。**

- 指定id创建文档

```
request:
PUT /test_index/doc/1
{
    "username": "alfred",
    "age": 1
}

response:
{
  "_index": "test_index",
  "_type": "doc",
  "_id": "1",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 0,
  "_primary_term": 1
}
```

- 不指定id创建文档

```
request:
POST /test_index/doc
{
    "username": "tom",
    "age": 20
}

response:
{
  "_index": "test_index",
  "_type": "doc",
  "_id": "FQQL8mUBmW-jRNGpqlAB",
  "_version": 1,
  "result": "created",
  "_shards": {
    "total": 2,
    "successful": 1,
    "failed": 0
  },
  "_seq_no": 0,
  "_primary_term": 1
}
```

- es允许一次创建多个文档，从而减少网络传输的开销，提升写入速率
  - endpoint为`_bulk`，如下：

```
request:
POST _bulk
{"index":{"_index":"test_index","_type":"doc","_id":"3"}}
{"username":"alfred","age":10}
{"delete":{"_index":"test_index","_type":"doc","_id":"1"}}
{"update":{"_id":"2","_index":"test_index","_type":"doc"}}
{"doc":{"age":"20"}}

response:
{
  "took": 74,
  "errors": true,
  "items": [
    {
      "index": {
        "_index": "test_index",
        "_type": "doc",
        "_id": "3",
        "_version": 1,
        "result": "created",
        "_shards": {
          "total": 2,
          "successful": 1,
          "failed": 0
        },
        "_seq_no": 0,
        "_primary_term": 1,
        "status": 201
      }
    },
    {
      "delete": {
        "_index": "test_index",
        "_type": "doc",
        "_id": "1",
        "_version": 2,
        "result": "deleted",
        "_shards": {
          "total": 2,
          "successful": 1,
          "failed": 0
        },
        "_seq_no": 1,
        "_primary_term": 1,
        "status": 200
      }
    },
    {
      "update": {
        "_index": "test_index",
        "_type": "doc",
        "_id": "2",
        "status": 404,
        "error": {
          "type": "document_missing_exception",
          "reason": "[doc][2]: document missing",
          "index_uuid": "wky5ufb3S5aDjom5I8HUtA",
          "shard": "2",
          "index": "test_index"
        }
      }
    }
  ]
}
```

说明："index"、"delete"、"update"等属于action_type，具体有如下几种：

- action_type
  - index 创建，如果已有则覆盖
  - update 更新
  - create 创建，如果已有则报错
  - delete 删除

### 5.2、查询文档

- 指定要查询的文档id

```
request:
GET /test_index/doc/1
成功的应答：
{
  "_index": "test_index",
  "_type": "doc",
  "_id": "1",
  "_version": 1,
  "found": true,
  "_source": {
    "username": "alfred",
    "age": 1
  }
}
失败的应答：
{
  "_index": "test_index",
  "_type": "doc",
  "_id": "2",
  "found": false
}
```

- 查询所有文档，用到`_search`，如下：

```
request:
GET /test_index/doc/_search
response:
{
  "took": 101,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 1,
    "hits": [
      {
        "_index": "test_index",
        "_type": "doc",
        "_id": "FQQL8mUBmW-jRNGpqlAB",
        "_score": 1,
        "_source": {
          "username": "tom",
          "age": 20
        }
      },
      {
        "_index": "test_index",
        "_type": "doc",
        "_id": "1",
        "_score": 1,
        "_source": {
          "username": "alfred",
          "age": 1
        }
      }
    ]
  }
}
```

```
request:
GET /test_index/doc/_search
{
    "query": {
        "term": {
        	"_id": "1"
        }
    }
}
response:
{
  "took": 19,
  "timed_out": false,
  "_shards": {
    "total": 5,
    "successful": 5,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 1,
    "hits": [
      {
        "_index": "test_index",
        "_type": "doc",
        "_id": "1",
        "_score": 1,
        "_source": {
          "username": "alfred",
          "age": 1
        }
      }
    ]
  }
}
```

- es允许一次查询多个文档
  - endpoint为`_mget`，如下：

```
GET /_mget
{
    "docs": [
        {
            "_index": "test_index",
            "_type": "doc",
            "_id": "1"
        },
        {
            "_index": "test_index",
            "_type": "doc",
            "_id": "2"
        }
    ]
}
response:
{
  "docs": [
    {
      "_index": "test_index",
      "_type": "doc",
      "_id": "1",
      "found": false
    },
    {
      "_index": "test_index",
      "_type": "doc",
      "_id": "3",
      "_version": 1,
      "found": true,
      "_source": {
        "username": "alfred",
        "age": 10
      }
    }
  ]
}
```

# 二、倒排索引与分词

## 1、正排索引与倒排索引简介

- 正排索引
  - 文档id到文档内容、单词的关联关系

| 文档ID | 文档内容                        |
| ------ | ------------------------------- |
| 1      | Elasticsearch是最流行的搜索引擎 |
| 2      | php是世界上最好的语言           |
| 3      | 搜索引擎是如何诞生的            |



- 倒排索引
  - 单词到文档id的关联关系

| 单词          | 文档ID列表 |
| ------------- | ---------- |
| Elasticsearch | 1          |
| 流行          | 1          |
| 搜索引擎      | 1,3        |
| php           | 2          |
| 世界          | 2          |
| 最好          | 2          |
| 语言          | 2          |
| 如何          | 3          |
| 诞生          | 3          |

- 查询包含“搜索引擎”的文档
  - 通过倒排索引获得“搜索引擎”对应的文档id有1和3
  - 通过正排索引查询1和3的完整内容
  - 返回用户最终结果

## 2、倒排索引的组成

- 倒排索引是搜索引擎的核心，主要包含两个部分：
  - 单词词典（Term Dictionary）
    - 是倒排索引的重要组成
    - 记录所有文档的单词，一般都比较大
    - 记录单词到倒排列表的关联信息
  - 倒排列表（Posting List）
    - 记录了单词对应文档的集合，由倒排索引项（Posting）组成
    - 倒排索引项（Posting）主要包含如下信息：
      - 文档id，用于获取原始信息
      - 单词频率（TF，Term Frequency），记录该单词在该文档中出现的次数，用于后续相关性算分
      - 位置（Position），记录单词在文档中的分词位置（多个），用于做词语搜索（Phrase Query）
      - 偏移（Offset），记录单词在文档的开始和结束位置，用于做高亮显示
- 示例：文档中“搜索引擎”这个词的分析

| 文档ID | 文档内容                        |
| ------ | ------------------------------- |
| 1      | Elasticsearch是最流行的搜索引擎 |
| 2      | php是世界上最好的语言           |
| 3      | 搜索引擎是如何诞生的            |

| DocId | TF   | Pofition | Offset  |
| ----- | ---- | -------- | ------- |
| 1     | 1    | 2        | <18,22> |
| 3     | 1    | 0        | <0,4>   |

![倒排索引](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180920075712.png)

- es存储的是一个json格式的文档，其中包含多个字段，每个字段会有自己的倒排索引，类似下图：

![字段的倒排索引](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180920080351.png)

## 3、分词

分词是指将文本转换成一系列单词（term or token）的过程，也可以叫做文本分析，在es里面称为Analysis，如下图所示：



- 分词器是es中专门处理分词的组件，英文为Analyzer，它的组成如下：
  - Character Filters
    - 针对原始文本进行处理，比如去除html特殊标记符
  - Tokenizer
    - 将原始文本按照一定规则切分为单词
  - Token Filter
    - 针对tokenizer处理的单词进行再加工，比如转小写、删除或者新增等处理

# 三、Mapping设置

# 四、Search API介绍

# 五、分布式特性介绍

# 六、深入了解Search的运行机制

# 七、聚合分析入门

# 八、数据建模

# 九、集群调优建议

