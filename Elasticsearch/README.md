# Elasticsearch

[TOC]

# 一、Elasticsearch入门

## 1、常见术语

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

## 2、Document

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

## 3、Index

- 索引中存储具有相同机构的文档（Document）
  - 每个索引都有自己的`mapping`定义，用于定义字段名和类型
- 一个集群可以有多个索引，比如：
  - nginx 日志存储的时候可以按照日期每天生成一个索引来存储
    - nginx-log-2017-01-01
    - nginx-log-2017-01-02
    - nginx-log-2017-01-03

## 4、Rest API

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

## 5、索引API

### 5.1、创建索引api如下：

```
PUT /test_index
```

### 5.2、查看现有索引

```
GET _cat/indices
```

### 5.3、删除索引

```
DELETE /test_index
```

## 6、文档 Document API

### 6.1、创建文档

**创建文档时，如果索引不存在，es会自动创建对应的index和type。**

- 指定id创建文档

```
PUT /test_index/doc/1
{
    "username": "alfred",
    "age": 1
}
```

- 不指定id创建文档

```
POST /test_index/doc
{
    "username": "tom",
    "age": 20
}
```

- es允许一次创建多个文档，从而减少网络传输的开销，提升写入速率
  - endpoint为`_bulk`，如下：

```
POST _bulk
{"index":{"_index":"test_index","_type":"doc","_id":"3"}}
{"username":"alfred","age":10}
{"delete":{"_index":"test_index","_type":"doc","_id":"1"}}
{"update":{"_id":"2","_index":"test_index","_type":"doc"}}
{"doc":{"age":"20"}}
```

说明："index"、"delete"、"update"等属于action_type，具体有如下几种：

- action_type
  - index 创建，如果已有则覆盖
  - update 更新
  - create 创建，如果已有则报错
  - delete 删除

### 6.2、查询文档

- 查询指定id的文档

```
GET /test_index/doc/1
```

- 查询所有文档，用到`_search`，如下：

```
GET /test_index/doc/_search
```

- 查询所有文档，用到`_search`，可以指定查询条件，如下：

```
GET /test_index/doc/_search
{
    "query": {
        "term": {
        	"_id": "1"
        }
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

![Analysis](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180920080909.png)

- 分词器是es中专门处理分词的组件，英文为Analyzer，它的组成如下：
  - Character Filters
    - 针对原始文本进行处理，比如去除html特殊标记符
  - Tokenizer
    - 将原始文本按照一定规则切分为单词
  - Token Filter
    - 针对tokenizer处理的单词进行再加工，比如转小写、删除或者新增等处理
- 分词器-调用顺序

![分词器-调用顺序](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180920081512.png)

## 4、Analyze API

- es提供了一个测试分词的api接口，方便验证分词效果，endpoint是_analyze
  - 可以直接指定analyzer进行测试
  - 可以直接指定索引中的字段进行测试
  - 可以自定义分词器进行测试


### 4.1、直接指定analyzer进行测试

```
POST _analyze
{
    "analyzer": "standard",
    "text": "hello world!"
}
```

### 4.2、直接指定索引中的字段（使用字段的分词器）进行测试

```
POST test_index/_analyze
{
    "field": "username",
    "text": "hello world!"
}
```

### 4.3、自定义分词器进行测试

```
POST _analyze
{
    "tokenizer": "standard",
    "filter": ["lowercase"],
    "text": "Hello World!"
}
```

## 5、预定义的分词器

测试语句：`The 2 QUICK Brown-Foxes jumped over the lazy dog's bone.`

### 5.1、Standard Analyzer

- `standard`
  - 默认分词器
  - 其组成如图，特性为：
    - 按词切分，支持多语言
    - 小写处理

```
POST _analyze
{
  "analyzer": "standard",
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
```

![Standard Analyzer](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180922075255.png)

### 5.2、Simple Analyzer

- `simple`
  - 其组成如图，特性为：
    - 按照非字母切分
    - 小写处理

```
POST _analyze
{
  "analyzer": "simple",
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
```

![Simple Analyzer](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180922075746.png)

###  5.3、Whitespace Analyzer

- `whitespace`
  - 其组成如图，特性为：
    - 按照空格切分

```
POST _analyze
{
  "analyzer": "whitespace",
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
```

![Whitespace Analyzer](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180922080108.png)

### 5.4、Stop Analyzer

- `stop`
  - Stop Word指语气助词等修饰性的词语，比如the、an、的、这等等
  - 其组成如图，特性为：
    - 相比Simple Analyzer多了Stop Word处理

```
POST _analyze
{
  "analyzer": "stop",
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
```

![Stop Analyzer](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180922080551.png)

### 5.5、Keyword Analyzer

- `keyword`
  - 其组成如图，特性为：
    - 不分词，直接将输入作为一个单词输出

```
POST _analyze
{
  "analyzer": "keyword",
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
```

![Keyword Analyzer](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180922081232.png)

### 5.6、Pattern Analyzer

- `pattern`
  - 其组成如图，特性为：
    - 通过正则表达式自定义分隔符
      - 默认是`\W+`，即非字词的符号作为分隔符

```
POST _analyze
{
  "analyzer": "pattern",
  "text": "The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."
}
```

![Pattern Analyzer](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180922083029.png)

### 5.7、Language Analyzer

- Language Analyzer
  - 提供了30+常见语言的分词器
  - `arabic`,`armenian`,`basque`,`bengali`,`brazilian`,`bulgarian`,`catalan`,`cjk`,`czech`,`danish`,`dutch`,`english`...

## 6、中文分词

- 难点
  - 中文分词指的是将一个汉字序列切分成一个一个单独的词。在英文中，单词之间是以空格作为自然分界符，汉语中词没有一个形式上的分界符。
  - 上下文不同，分词结果迥异，比如交叉歧义问题，比如下面两种分词都合理
    - 乒乓球拍/卖/完了
    - 乒乓球/拍卖/完了
- 常用分词系统
  - IK
    - 实现中英文单词的切分，支持ik_smart、ik_maxword等模式
    - 可自定义词库，支持热更新分词词典
    - https://github.com/medcl/elasticsearch-analysis-ik
  - jieba
    - python中最流行的分词系统，支持分词和词性标注
    - 支持繁体分词、自定义词典、并行分词等
    - https://github.com/sing1ee/elasticsearch-jieba-plugin
- 基于自然语言处理的分词系统
  - Hanlp
    - 由一系列模型与算法组成的Java工具包，目标是普及自然语言处理在生产环境中的应用
    - https://github.com/hankcs/HanLP
  - THULAC
    - THU Lexical Analyzer for Chinese，由清华大学自然语言处理与社会人文计算实验室研制推出的一套中文词法分析工具包，具有中文分词和词性标注功能
    - https://github.com/microbun/elasticsearch-thulac-plugin

## 7、自定义分词

- 当自带的分词无法满足需求时，可以自定义分词
  - 通过自定义Character Filters、Tokenizer和Token Filter实现
- Character Filters
  - 在Tokenizer之前对原始文本进行处理，比如增加、删除或替换字符等
  - 自带的如下：
    - HTML Strip去除html标签和转换html实体
    - Mapping进行字符替换操作
    - Pattern Replace进行正则匹配替换
  - 会影响后续tokenizer解析的postion和offset信息
  - 测试举例：

```
request:
POST _analyze
{
    "tokenizer": "keyword",
    "char_filter": ["html_strip"],
    "text": "<p>I&apos;m so <b>happy</b>!</p>"
}
response:
{
  "tokens": [
    {
      "token": """

I'm so happy!

""",
      "start_offset": 0,
      "end_offset": 32,
      "type": "word",
      "position": 0
    }
  ]
}
```

- Tokenizer
  - 将原始文本按照一定规则切分为单词（term or token）
  - 自带的如下：
    - standard 按照单词进行分割
    - letter 按照非字符类进行分割
    - whitespace 按照空格进行分割
    - UAX URL Email 按照standard分割，但不会分割邮箱和url
    - NGram和Edge NGram连词分割
    - Path Hierarchy 按照文件路径进行分割
    - 测试举例：

```
request:
POST _analyze
{
    "tokenizer": "path_hierarchy",
    "text": "/one/two/three"
}
response:
{
  "tokens": [
    {
      "token": "/one",
      "start_offset": 0,
      "end_offset": 4,
      "type": "word",
      "position": 0
    },
    {
      "token": "/one/two",
      "start_offset": 0,
      "end_offset": 8,
      "type": "word",
      "position": 0
    },
    {
      "token": "/one/two/three",
      "start_offset": 0,
      "end_offset": 14,
      "type": "word",
      "position": 0
    }
  ]
}
```

- Token Filters
  - 对于tokenizer输出的单词（term）进行增加、删除、修改等操作
  - 自带的如下：
    - lowercase 将所有term转换为小写
    - stop 删除stop words
    - NGram和Edge NGram连词分割
    - Synonym 添加近义词的 term

```
request:
POST _analyze
{
    "text": "a Hello,world!",
    "tokenizer": "standard",
    "filter": [
        "stop",
        "lowercase",
        {
            "type": "ngram",
            "min_gram": 4,
            "max_gram": 4
        }
    ]
}
response:
{
  "tokens": [
    {
      "token": "hell",
      "start_offset": 2,
      "end_offset": 7,
      "type": "<ALPHANUM>",
      "position": 1
    },
    {
      "token": "ello",
      "start_offset": 2,
      "end_offset": 7,
      "type": "<ALPHANUM>",
      "position": 1
    },
    {
      "token": "worl",
      "start_offset": 8,
      "end_offset": 13,
      "type": "<ALPHANUM>",
      "position": 2
    },
    {
      "token": "orld",
      "start_offset": 8,
      "end_offset": 13,
      "type": "<ALPHANUM>",
      "position": 2
    }
  ]
}
```

## 8、自定义分词的API

- 自定义分词的API
  - 自定义分词需要在索引的配置中设定，如下所示：

```
PUT test_index
{
    "settings": {
    	"analysis": {
            "char_filter": {},
            "tokenizer": {},
            "filter": {},
            "analyzer": {}
    	}
    }
}
```

### 8.1、简单示例

- 定义

```
PUT test_index
{
    "settings": {
        "analysis": {
            "analyzer": {
                "my_custom_analyzer": {
                    "type": "custom",
                    "tokenizer": "standard",
                    "char_filter": [
                        "html_strip"
                    ],
                    "filter": [
                        "lowercase",
                        "asciifolding"
                    ]
                }
            }
        }
    }
}
```

- 验证

```
POST test_index/_analyze
{
    "analyzer": "my_custom_analyzer",
    "text": "Is this <b>a box</b>?"
}
```

### 8.2、复杂示例

- 定义

```
PUT test_index2
{
	"settings": {
        "analysis": {
            "analyzer": {
                "my_custom_analyzer": {
                    "type": "custom",
                    "char_filter": [
                    	"emoticons"
                    ],
                    "tokenizer": "punctuation",
                    "filter": [
                        "lowercase",
                        "english_stop"
                    ]
                }
            },
            "tokenizer": {
                "punctuation": {
                    "type": "pattern",
                    "pattern": "[.,!?]"
                }
            },
            "char_filter": {
                "emoticons": {
                    "type": "mapping",
                    "mappings": [
                        ":) => _happy_",
                        ": => _sad_"
                    ]
                }
            },
            "filter": {
                "english_stop": {
                    "type": "stop",
                    "stopwords": "_english_"
                }
            }
        }
	}
}
```

- 验证

```
POST test_index2/_analyze
{
    "analyzer": "my_custom_analyzer",
    "text": "I'm a :) person, and you?"
}
```

## 9、分词使用说明

- 分词会在如下两个时机使用：

  - 创建或更新文档时（Index Time），会对相应的文档进行分词处理
  - 查询时（Search Time），会对查询语句进行分词

- 索引时分词是通过配置Index Mapping中每个字段的analyzer属性实现的，如下：

  - 不指定分词时，使用默认standard

  ```
  PUT test_index
  {
      "mappings": {
          "doc": {
              "properties": {
                  "title": {
                      "type": "text",
                      "analyzer": "whitespace"
                  }
              }
          }
      }
  }
  ```

- 查询时分词的指定方式有如下的几种：

  - 查询的时候通过analyzer指定分词器

  ```
  POST test_index/_search
  {
      "query": {
          "match": {
              "message": {
                  "query": "hello",
                  "analyzer": "standard"
              }
          }
      }
  }
  ```

  - 通过index mapping设置search_analyzer实现

  ```
  PUT test_index
  {
      "mappings": {
          "doc": {
              "properties": {
                  "title": {
                      "type": "text",
                      "analyzer": "whitespace",
                      "search_analyzer": "standard"
                  }
              }
          }
      }
  }
  ```

**一般不需要特别指定查询时分词器，直接使用索引时分词器即可，否则会出现无法匹配的情况。**

- 分词的使用建议
  - 明确字段是否需要分词，不需要分词的字段就将type设置为keyword，可以节省空间和提高性能
  - 善用`_analyze API`，查看文档的具体分词结果
  - 动手测试

# 三、Mapping设置

## 1、Mapping简介

- 类似数据库中的表结构定义，主要作用如下：
  - 定义Index下的字段名（Field Name）
  - 定义字段的类型，比如数值型、字符串型、布尔型等
  - 定义倒排索引相关的配置，比如是否索引、记录position等

查询Mapping：

```
# 查询mappings
GET /test_index/_mapping
# 查询settings
GET /test_index/_settings
# 查询索引
GET /test_index
```

## 2、自定义Mapping

- 自定义Mapping的api如下所示：

```
PUT my_index
{
    "mappings": {
        "doc": {
        	"properties": {
                "title": {
                    "type": "text"
                },
                "name": {
                    "type": "keyword"
                },
                "age": {
                    "type": "integer"
                }
        	}
        }
    }
}
```

- Mapping中的字段类型一旦设定后，禁止直接修改，原因如下：
  - Lucene实现的倒排索引生成后不允许修改
- 重新建立新的索引，然后做reindex操作
- 允许新增字段
- 通过dynamic参数来控制字段的新增
  - true （默认）允许自动新增字段
  - false 不允许自动新增字段，但是文档可以正常写入，但无法对字段进行查询等操作
  - strict 文档不能写入，报错

```
PUT my_index
{
    "mappings": {
        "my_type": {
            "dynamic": false,
            "properties": {
                "user": {
                    "properties": {
                        "name": {
                            "type": "text"
                        },
                        "social_networks": {
                            "dynamic": true,
                            "properties": {}
                        }
                    }
                }
            }
        }
    }
}
```

### copy_to

- 将该字段的值复制到目标字段，实现类似`_all`的作用
- 不会出现在`_source`中，只用来搜索

```
# 创建索引
PUT my_index
{
    "mappings": {
        "doc": {
            "properties": {
                "first_name": {
                    "type": "text",
                    "copy_to": "full_name"
                },
                "last_name": {
                    "type": "text",
                    "copy_to": "full_name"
                },
                "full_name": {
                    "type": "text"
                }
            }
        }
    }
}
```

```
# 创建文档
PUT my_index/doc/1
{
    "first_name": "John",
    "last_name": "Smith"
}
```

```
# 查询文档
GET my_index/_search
{
    "query": {
        "match": {
            "full_name": {
                "query": "John Smith",
                "operator": "and"
            }
        }
    }
}
```

###  index

- 控制当前字段是否索引，默认为true，即记录索引，false不记录，即不可搜索

```
# 创建索引
PUT my_index
{
    "mappings": {
        "doc": {
            "properties": {
                "cookie": {
					"type": "text",
					"index": false
                }
            }
        }
    }
}
```

```
# 查询文档
GET my_index/_search
{
    "query": {
        "match": {
        	"cookie": "name"
        }
    }
}
```

### index_options

- 用于控制倒排索引记录的内容，有如下4种配置
  - `docs` 只记录doc id
  - `freqs` 记录doc id和term frequencies
  - `positions` 记录doc id、term frequencies和term position
  - `offsets` 记录doc id、term frequencies、term position和character offsets
- text类型默认配置为positions，其他默认为docs
- 记录内容越多，占用空间越大

```
# 创建索引
PUT my_index
{
    "mappings": {
        "doc": {
            "properties": {
                "cookie": {
                    "type": "text",
                    "index_options": "offsets"
                }
            }
        }
    }
}
```

### null_value

- 当字段遇到null值时的处理策略，默认为null，即空值，此时es会忽略该值。可以通过设定该值设定字段的默认值

```
# 创建索引
PUT my_index
{
    "mappings": {
        "my_type": {
            "properties": {
                "status_code": {
                    "type": "keyword",
                    "null_value": "NULL"
                }
            }
        }
    }
}
```

### 核心数据类型

- 字符串型 text、keyword
- 数值型 long、integer、short、byte、double、float、half_float、scaled_float
- 日期类型 date
- 布尔类型 boolean
- 二进制类型 binary
- 范围类型 integer_range、float_range、long_range、double_range、date_range
- 复杂数据类型
  - 数组类型 array
  - 对象类型 object 
  - 嵌套类型 nested object
- 地理位置数据类型
  - geo_point
  - geo_shape
- 专用类型
  - 记录IP地址 `ip`
  - 实现自动补全 `completion`
  - 记录分词数 `token_count`
  - 记录字符串hash值 `murmur3`
  - percolator
  - join

### 多字段特性

- 多字段特性 multi-fields

  - 允许对同一个字段采用不同的配置，比如分词，常见例子如对人名实现拼音搜索，只需要在人名中新增一个子字段为pinyin即可。

  ```
  {
      "test_index": {
          "mappings": {
          	"doc": {
                  "properties": {
                      "username": {
                          "type": "text",
                          "fields": {
                              "pinyin": {
                                  "type": "text",
                                  "analyzer": "pinyin"
                              }
                          }
                      }
                  }
          	}
          }
      }
  }
  ```

  ```
  GET test_index/_search
  {
      "query": {
      	"match": {
              "username.pinyin": "hanhan"
      	}
      }
  }
  ```

### Dynamic Mapping

- es可以自动识别文档字段类型，从而降低用户使用成本，如下所示：

```
PUT /test_index/doc/1
{
    "username": "alfred",
    "age": 1
}
```

```
GET /test_index/_mapping
```

- es是依靠JSON文档的字段类似来实现自动识别字段类型，支持的类型如下：

| JSON类型 | es类型                                                       |
| -------- | ------------------------------------------------------------ |
| null     | 忽略                                                         |
| boolean  | boolean                                                      |
| 浮点类型 | float                                                        |
| 整数     | long                                                         |
| object   | object                                                       |
| array    | 由第一个非null值类型决定                                     |
| string   | 匹配为日期则设为date类型（默认开启）<br />匹配为数字的话设为float或long类型（默认关闭）<br />设为text类型，并附带keyword的子字段 |

```
PUT /test_index/doc/1
{
    "username": "alfred",
    "age": 14,
    "birth": "1988-10-10",
    "married": false,
    "year": "18",
    "tags": ["boy", "fashion"],
    "money": 100.1
}
```

```
GET /test_index/_mapping
```

- 日期的自动识别可以自行配置日期格式，以满足各种需求

  - 默认是`["strict_date_optional_time", "yyyy/MM/dd HH:mm:ss Z||yyyy/MM/dd Z"]`
  - strict_date_optional_time是ISO datetime格式，完整格式类似下面：
    - YYYY-MM-DDThh:mm:ssTZD(eg 1997-07-16T19:20:30+01:00)
  - dynamic_date_formats可以自定义日期类型
  - date_detection可以关闭日期自动识别的机制

- 字符串是数字时，默认不会自动识别为整形，因为字符串中出现数字是完全合理的

  - numeric_detection可以开启字符串中数字的自动识别，如下所示：

  ```
  PUT my_index
  {
      "mappings": {
          "my_type": {
              "numeric_detection": true
          }
      }
  }
  ```

### Dynamic Templates

- 允许根据es自动识别的数据类型、字段名等来动态设定字段类型，可以实现如下效果：
  - 所有字符串类型都设置为keywork类型，即默认不分词
  - 所有以message开头的字段都设置为text类型，即分词
  - 所有以long_开头的字段都设定为long类型
  - 所有自动匹配为double类型的都设定为float类型，以节省空间

- 匹配规则一般有如下几个参数：
  - match_mapping_type 匹配es自动识别的字段类型，如boolean、long、string等
  - match,unmatch 匹配字段名
  - path_match,patch_unmatch 匹配路径

```
# 创建索引
PUT test_index
{
    "mappings": {
        "doc": {
            "dynamic_templates": [
                {
                    "message_as_text": {
                        "match_mapping_type": "string",
                        "match": "message*",
                        "mapping": {
                            "type": "text"
                        }
                    }
                },
                {
                    "strings_as_keywords": {
                        "match_mapping_type": "string",
                        "mapping": {
                            "type": "keyword"
                        }
                    }
                },
                {
                    "double_as_float": {
                        "match_mapping_type": "double",
                        "mapping": {
                            "type": "float"
                        }
                    }
                }
            ]
        }
    }
}
```

```
# 创建文档
PUT test_index/doc/1
{
    "name": "alfred",
    "message": "handsome boy"
}
```

### 自定义Maping的建议

自定义Mapping的操作步骤如下：

1. 写入一条文档到es的临时索引中，获取es自动生成的mapping

```
# 示例
PUT test_index/doc/1
{
    "referrer": "-",
    "response_code": "200",
    "remote_ip": "171.221.139.157",
    "method": "POST",
    "user_name": "-",
    "http_version": "1.1",
    "body_sent": {
        "bytes": "0"
    },
    "url": "/analyzeVideo"
}
```

2. 修改步骤1得到的mapping，自定义相关配置

```
# 获取并修改
GET /test_index/_mapping
```

3. 使用步骤2的mapping创建实际所需索引

### 索引模板

- 索引模板，英文为Index Template，主要用于在创建索引时自动应用预先设定的配置，简化索引创建的操作步骤
  - 可以设定索引的配置和mapping
  - 可以有多个模板，根据order设置，order大的覆盖小的配置
- 索引模板API，endpoint为`_template`，如下所示：

```
# 模板1
PUT _template/test_template
{
    "index_patterns": ["te*", "bar*"],
    "order": 0,
    "settings": {
        "number_of_shards": 1
    },
    "mappings": {
        "doc": {
			"_source": {
                "enabled": false
			},
			"properties": {
                "name": {
                    "type": "keyword"
                }
			}
        }
    }
}

# 模板2
PUT _template/test_template2
{
	"index_patterns": ["test*"],
	"order": 1,
	"settings": {
        "number_of_shards": 1
	},
	"mappings": {
        "doc": {
            "_source": {
                "enabled": true
            }
        }
	}
}
```

```
# 创建索引并查看配置
PUT test_index
GET test_index
```

- 获取与删除索引模板

```
GET _template
GET _template/test_template
DELETE _template/test_template
```

# 四、Search API介绍

## 1、Search API

- 实现对es中存储的数据进行查询分析，endpoint为`_search`，如下所示：

```
GET /_search
GET /my_index/_search
GET /my_index1,my_index2/_search
GET /my_*/_search
```

- 查询主要有两种形式

  - URI Search
    - 操作简便，方便通过命令行测试
    - 仅包含部分查询语法

  ```
  GET /my_index/_search?q=user:alfred
  ```

  - Request Body Search
    - es提供的完备查询语法Query DSL（Domain Specific Language）

  ```
  GET /my_index/_search
  {
      "query": {
          "term": {"user": "alfred"}
      }
  }
  ```

- 数据准备

```
POST test_search_index/doc/_bulk
{
    "index": {
        "_id": "1"
    }
}
{
    "username": "alfred way",
    "job": "java engineer",
    "age": 18,
    "birth": "1990-01-02",
    "isMarried": false
}
{
    "index": {
        "_id": "2"
    }
}
{
    "username": "alfred",
    "job": "java senior engineer and java specialist",
    "age": 28,
    "birth": "1980-05-07",
    "isMarried": true
}
{
    "index": {
        "_id": "3"
    }
}
{
    "username": "lee",
    "job": "java and ruby engineer",
    "age": 22,
    "birth": "1985-08-07",
    "isMarried": false
}
{
    "index": {
        "_id": "4"
    }
}
{
    "username": "alfred junior way",
    "job": "ruby engineer",
    "age": 23,
    "birth": "1980-08-07",
    "isMarried": false
}
```

## 2、URI Search

- 通过 url query 参数来实现搜索，常用参数如下：
  - `q` 指定查询的语句，语法为Query String Syntax
  - `df` q中不指定字段时默认查询的字段，如果不指定，es会查询所有字段
  - `sort` 排序
  - `timeout` 指定超时时间，默认不超时
  - `from,size` 用于分页

```
# 查询user字段包含alfred的文档，结果按照age升序排列，返回第5-14个文档，如果超过1s没有结束，则以超时结束。
GET /my_index/_search?q=alfred&df=user&sort=age:asc&from=4&size=10&timeout=1s
```

### Query String Syntax

- term 与 phrase（单次term与词语查询）
  - `alfred way`等效于alfred OR way
  - `"alfred way"` 词语查询，要求先后顺序
- 泛查询
  - `alfred` 等效于在所有字段去匹配该term
- 指定字段
  - `name:alfred`
- Group分组设定，使用括号指定匹配的规则
  - (quick OR brown) AND fox
  - status:(active OR pending) title:(full text search)

```
# 如果想看执行细节，可以添加profile配置
GET test_search_index/_search?q=alfred
{
  "profile": "true"
}
GET test_search_index/_search?q=username:alfred
GET test_search_index/_search?q=username:alfred way
GET test_search_index/_search?q=username:"alfred way"
GET test_search_index/_search?q=username:(alfred way)
```

- 布尔操作符
  - AND(&&), OR(||), NOT(!)
    - name:(tom NOT lee)
    - 注意大写，不能小写
  - `+`和`-`分别对应must和must_not，且`+`或`-`要与后面内容紧贴着
    - name:(tom `+`lee `-`alfred)
    - name:((lee && !alfred) || (tom && lee && !alfred))
    - `+`在url中会被解析为空格，要使用encode后的结果才可以，为`%2B`

```
GET test_search_index/_search?q=username:alfred AND way
GET test_search_index/_search?q=username:(alfred AND way)
GET test_search_index/_search?q=username:(alfred NOT way)
GET test_search_index/_search?q=username:(alfred %2Bway)
```

- 范围查询，支持数值和日期
  - 区间写法，闭区间用[]，开区间用{}
    - age:[1 TO 10] 意为 1 <= age <= 10
    - age:[1 TO 10} 意为 1<= age < 10
    - age:[1 TO] 意为 age >= 1
    - age:[* TO 10] 意为 age <= 10
  - 算数符号写法
    - age:>=1
    - age:(>=1 && <=10) 或者age:(+>=1 +<=10)

```
GET test_search_index/_search?q=username:alfred age:>20
GET test_search_index/_search?q=birth:(>1980 AND <1990)
```

- 通配符查询
  - `?` 代表1个字符，*代表0或多个字符
    - name:t?m
    - name:tom*
    - name:t*m
  - 通配符匹配执行效率低，且占用较多内存，不建议使用
  - 如无特殊需求，不要将?/*放在最前面

```
GET test_search_index/_search?q=username:alf*
```

- 正则表达式匹配
  - name:/[mb]oat/

```
GET test_search_index/_search?q=username:/[a]?l.*/
```

- 模糊匹配 fuzzy query
  - name:roam~1
  - 匹配与roam相差一个character的词，比如form roams等
- 近似度查询 proximity search
  - "fox quick"~5
  - 以term为单位进行差异比较，比如"quick fox","quick brown fox"都会被匹配

```
GET test_search_index/_search?q=username:alfed~1
GET test_search_index/_search?q=username:alfd~2
GET test_search_index/_search?q=job:"java engineer"~1
```

## 3、Request Body Search

- 将查询语句通过http request body发送到es，主要包含如下参数
  - `query` 符合Query DSL语法的查询语句
  - `from`,`size`
  - `timeout`
  - `sort`
  - ...
- 基于JSON定义的查询语言，主要包含如下两种类型：
  - 字段类查询
    - 如`term`,`match`,`range`等，只针对某一个字段进行查询
  - 复合查询
    - 如bool查询等，包含一个或多个字段类查询或者复合查询语句

### 字段类查询

- 字段类查询主要包括以下两类：

  - 全文匹配
    - 针对text类型的字段进行全文检索，会对查询语句进行**分词**处理，如match，match_phrase等query类型
  - 单词匹配
    - 不会对查询语句做分词处理，直接去匹配字段的倒排索引，如term，terms，range等query类型

- Match Query

  - 对字段作全文检索，最基本和常用的查询类型，API示例如下：

  ```
  # 如果想看执行细节，可以添加profile配置
  GET test_search_index/_search
  {
  	"profile": "true",
      "query": {
          "match": {
              "username": "alfred way"
          }
      }
  }
  ```

  - 通过`operator`参数可以控制单词间的匹配关系，可选项为`or`和`and`

  ```
  GET test_search_index/_search
  {
  	"profile": "true",
      "query": {
          "match": {
              "username": {
                  "query": "alfred way",
                  "operator": "and"
              }
          }
      }
  }
  ```

  - 通过`minimum_should_match`参数可以控制需要匹配的单词数

  ```
  GET test_search_index/_search
  {
  	"profile": "true",
      "query": {
          "match": {
              "username": {
                  "query": "alfred way",
                  "minimum_should_match": "2"
              }
          }
      }
  }
  ```

- Match Phrase Query

  - 对字段作检索，有顺序要求，API示例如下

  ```
  GET test_search_index/_search
  {
      "query": {
          "match_phrase": {
              "job": "java engineer"
          }
      }
  }
  ```

  - 通过slop参数可以控制单词间的间隔

  ```
  
  GET test_search_index/_search
  {
      "query": {
          "match_phrase": {
              "job": {
                  "query": "java engineer",
                  "slop": "1"
              }
          }
      }
  }
  ```

- Query String Query

  - 类似于URI Search中的q参数查询

  ```
  GET test_search_index/_search
  {
      "query": {
          "query_string": {
              "default_field": "username",
              "query": "alfred AND way"
          }
      }
  }
  ```

  ```
  GET test_search_index/_search
  {
      "query": {
          "query_string": {
              "fields": ["username", "job"],
              "query": "alfred OR (java AND ruby)"
          }
      }
  }
  ```

- Simple Query String Query
  - 类似Query String，但是会忽略错误的查询语法，并且仅支持部分查询方法
  - 其常用的逻辑符号如下，不能使用AND、OR、NOT等关键词
  - `+` 代指 AND
  - `|` 代指OR
  - `-` 代指NOT

```
GET test_search_index/_search
{
    "query": {
        "simple_query_string": {
            "query": "alfred +way",
            "fields": ["username"]
        }
    }
}
```

- Term Query
  - 将查询语句作为整个单词进行查询，即不对查询语句做分词处理，如下所示：

```
GET test_search_index/_search
{
    "query": {
        "term": {
            "username": "alfred"
        }
    }
}
```

```
GET test_search_index/_search
{
    "query": {
        "terms": {
            "username": [
                "alfred",
                "way"
            ]
        }
    }
}
```

- Range Query

  - 范围查询主要针对数值和日期类型，如下所示：

  ```
  GET test_search_index/_search
  {
      "query": {
          "range": {
              "age": {
                  "gte": 10,
                  "lte": 20
              }
          }
      }
  }
  ```

  - 针对日期做查询，如下所示：

  ```
  GET test_search_index/_search
  {
      "query": {
          "range": {
              "birth": {
                  "gte": "1990-01-01"
              }
          }
      }
  }
  ```

  - 针对日期提供的一种更友好地计算方式，格式如下：

    - 单位主要有如下几种：

      - `y` years
      - `M` months
      - `w` weeks
      - `d` days
      - `h` hours
      - `m` minutes
      - `s` seconds

    - 假设now为2018-01-02 12:00:00，那么如下的计算结果实际为：

      | 计算公式            | 实际结果            |
      | ------------------- | ------------------- |
      | now + 1h            | 2018-01-02 13:00:00 |
      | now - 1h            | 2018-01-02 11:00:00 |
      | now -1h/d           | 2018-01-02 00:00:00 |
      | 2016-01-01\|\|+1M/d | 2016-02-01 00:00:00 |

  ```
  GET test_search_index/_search
  {
      "query": {
          "range": {
              "birth": {
                  "gte": "now-20y"
              }
          }
      }
  }
  ```

### 复合查询

- 复合查询是指包含字段类查询或复合查询的类型，主要包括以下几类：

  - `constant_score` query
  - `bool` query
  - `dis_max` query
  - `function_score` query
  - `boosting` query

- Constant Score Query

  - 该查询将其内部的查询结果文档得分都设定为1或boost的值
    - 多用于结合bool查询实现自定义得分

  ```
  GET test_search_index/_search
  {
      "query": {
          "constant_score": {
              "filter": {
                  "match": {
                      "username": "alfred"
                  }
              }
          }
      }
  }
  ```

- Bool Query

  - 布尔查询由一个或多个布尔子句组成，主要包含如下4个：

  | 关键字   | 描述                                           |
  | -------- | ---------------------------------------------- |
  | filter   | 只过滤符合条件的文档，不计算相关性得分         |
  | must     | 文档必须符合must中的所有条件，会影响相关性得分 |
  | must_not | 文档必须不符合must_not中所有的条件             |
  | should   | 文档可以符合should中的条件，会影响相关性得分   |

  - Bool查询的API如下所示：

  ```
  GET test_search_index/_search
  {
      "query": {
          "bool": {
              "must": [
                  {}
              ],
              "must_not": [
                  {}
              ],
              "should": [
                  {}
              ],
              "filter": [
              	{}
              ]
          }
      }
  }
  ```

  - Filter 查询只过滤符合条件的文档，不会进行相关性算分
    - es 针对filter会有智能缓存，因此其执行效率很高
    - 做简单匹配查询且不考虑算分时，推荐使用filter替代query等

  ```
  GET test_search_index/_search
  {
      "query": {
          "bool": {
              "filter": [
                  {
                      "term": {
                          "username": "alfred"
                      }
                  }
              ]
          }
      }
  }
  ```

  - Must

  ```
  GET test_search_index/_search
  {
      "query": {
          "bool": {
          	"must": [
                  {
                      "match": {
                      	"username": "alfred"
                  	}
                  },
                  {
                      "match": {
                          "job": "specialist"
                      }
                  }
          	]
          }
      }
  }
  ```

  - Must_Not

  ```
  GET test_search_index/_search
  {
      "query": {
          "bool": {
          	"must": [
                  {
                      "match": {
                      	"job": "java"
                  	}
                  }
          	],
          	"must_not": [
                  {
                      "match": {
                          "job": "ruby"
                      }
                  }
          	]
          }
      }
  }
  ```

  - Should

    - 只包含should时，文档必须满足至少一个条件
      - `minimum_should_match`可以控制满足条件的个数或者百分比

    ```
    GET test_search_index/_search
    {
        "query": {
            "bool": {
                "should": [
                    {"term": {"job": "java"}},
                    {"term": {"job": "ruby"}},
                    {"term": {"job": "specialist"}}
                ],
                "minimum_should_match": 2
            }
        }
    }
    ```

    - 同时包含should和must时，文档不必满足should中的条件，但是如果满足条件，会增加相关性得分

    ```
    # 查询username包含alfred的文档，同时将job包含ruby的文档排在前面
    GET test_search_index/_search
    {
        "query": {
            "bool": {
                "must": [
                	{"term": {"username": "alfred"}}
                ],
                "should": [
                    {"term": {"job": "ruby"}}
                ]
            }
        }
    }
    ```

  - Query Context VS Filter Context

    - 当一个查询语句位于Query或者Filter上下文时，es执行的结果会不同，对比如下：

    | 上下文类型 | 执行类型                                                     | 使用方式                                                 |
    | ---------- | ------------------------------------------------------------ | -------------------------------------------------------- |
    | Query      | 查找与查询语句最匹配的文档，<br />对所有文档进行相关性算分并排序 | query<br />bool中的must和should                          |
    | Filter     | 查找与查询语句相匹配的文档                                   | bool中农的filter与must_not<br />constant_score中的filter |


### 相关性算分

- 相关性算分是指文档与查询语句间的相关度，英文为relevance

  - 通过倒排索引可以获取与查询语句相匹配的文档列表，那么**如何将最符合用户查询需求的文档放到前列呢？**

  - 本质是一个排序问题，排序的依据是相关性算分

    ​								倒排索引

| 单词   | 文档ID列表 |
| ------ | ---------- |
| alfred | 1,2        |
| way    | 1          |

- 可以通过`explain`参数来查看具体的计算方法，但要注意：

  - es的算分是按照shard进行的，即shard的分数计算是相互独立的，所以在使用`explain`的时候注意分片数

  ```
  GET test_search_index/_search
  {
      "explain": true,
      "query": {
          "match": {
              "username": "alfred way"
          }
      }
  }
  ```

  - 可以通过设置索引的分片数为1来避免这个问题

  ```
  PUT test_search_index
  {
      "settings": {
          "index": {
              "number_of_shards": "1"
          }
      }
  }
  ```

### Count API

- 获取符合条件的文档数，endpoint为`_count`

```
GET test_search_index/_count
{
    "query": {
    	"match": {
            "username": "alfred"
    	}
    }
}
```

### Source Filtering

- 过滤返回结果中`_source`中的字段，主要有如下几种方式：

  - url参数

  ```
  GET test_search_index/_search?_source=username
  ```

  -  不返回`_source`

  ```
  GET test_search_index/_search
  {
      "_source": false
  }
  ```

  - 返回部分字段之一

  ```
  GET test_search_index/_search
  {
      "_source": {
      	"includes": "*i*",
      	"excludes": "birth"
      }
  }
  ```

  - 返回部分字段之二

  ```
  GET test_search_index/_search
  {
      "_source": ["username", "age"]
  }
  ```


# 五、分布式特性介绍

- es支持集群模式，是一个分布式系统，期好处主要有两个：
  - 增大系统容量，如内存、磁盘、使得es集群可以支持PB级的数据
  - 提高系统可用性，即时部分节点停止服务，整个集群依然可以正常服务
- es集群由多个es实例组成
  - 不同集群通过集群名字来区分，可以通过cluster.name进行修改，默认为elasticsearch
  - 每个es实例本质上是一个JVM进程，且有自己的名字，通过node.name进行修改

## 1、Master Node

- 可以修改cluster state的节点称为master节点，一个集群只能有一个
- cluster state存储在每个节点上，master维护最新版本并同步给其他节点
- master节点是通过集群中所有节点选举产生的，可以被选举的节点称为master-eligible节点，相关配置如下：
  - `node.master:true`

## 2、Coordinating Node

- 处理请求的节点即为coordinating节点，该节点为所有节点的默认角色，不能取消
  - 路由请求到正确的节点处理，比如创建索引的请求到master节点

## 3、Data Node

- 存储数据的节点即为data节点，默认节点都是data类型，相关配置如下：
  - `node.data:true`

## 4、提高系统可用性

- 服务可用性
  - 2个节点的情况下，允许其中一个节点停止服务
- 数据可用性
  - 引入副本（Replication）解决
  - 每个节点上都有完备的数据

## 5、增大系统容量

- 如何将数据分布于所有节点上？
  - 引入分片（Shard）解决问题
- 分片是es支持PB级数据的基石
  - 分片存储了部分数据，可以分布于任意节点上
  - 分片数在索引创建时指定且后续不允许再更改，默认5个
  - 分片有主分片和副本分片之分，以实现数据的高可用
  - 副本分片的数据由主分片同步，可以有多个，从而提高读取的吞吐量
- 3个分片和1个副本的示例：

```
PUT test_index
{
    "settings": {
        "number_of_shards": 3,
        "number_of"replicas": 1
    }
}
```

- 基于3个分片1个备份，如果已经有3个节点的情况下进行分析
  - 此时增加节点是否能提高test_index的数据容量？
    - 不能。因为只有3个分片，已经分布在3台节点上，新增的节点无法利用。
  - 此时增加副本数是否能提高test_index的读取吞吐量？
    - 不能。因为新增的副本也是分布在这3个节点上，还是利用了同样的资源。如果要增加吞吐量，还需要新增节点。
  - 结论：分片数的设定很重要，需要提前规划好
    - 过小会导致后续无法通过增加节点实现水平扩容
    - 过大会导致一个节点上分布过多分片，造成资源浪费，同时会影响查询性能

## 6、集群状态 Cluster Health

- 通过如下api可以查看集群健康状况，包括以下三种：
  - green 健康状态，指所有主副分片都正常分配
  - yellow 指所有主分片都正常分配，但是有副本分片未正常分配
  - red 有主分片未分配

```
GET _cluster/health
```

## 7、故障转移

- 集群由3个节点组成，如下所示，此时集群状态是green

- node1所在机器宕机导致服务终止，此时集群会如何处理？

  - node2和node3发现node1无法响应一段时间后会发起master选举，比如这里选择node2为master节点。此时由于主分片P0下线，集群状态变为Red。

  ![第一步](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180927124418.png)

  - node2发现主分片P0未分配，将R0提升为主分片。此时由于所有主分片都正常分配，集群状态变为Yellow。

  ![第二部](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180927124101.png)

  - node2为P0和P1生成新的副本，集群状态变为绿色。

  ![第三步](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180927124109.png)

## 8、文档分布式存储

- 文档最终会存储在分片上，如下图所示：

  - Document1最终存储在分片P1上

  ![存储在P1分片上](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180927132700.png)

- Document1是如何存储到分片P1的？选择P1的依据是什么？

  - 需要文档到分片的映射算法

- 目的

  - 使得文档均匀分布在所有分片上，以充分利用资源

- 算法

  - 随机选择或者round-robin算法？
    - 不可取，因为需要维护文档到分片的映射关系，成本巨大
  - 根据文档值实时计算对应的分片

### 文档到分片的映射算法

- es通过如下的公式计算文档对应的分片
  - shard = hash(routing)%number_of_primary_shards
    - hash算法保证可以将数据均匀分散在分片中
    - routing是一个关键参数，默认是文档的id，也可以自行制定
    - number_of_primary_shards是主分片数
  - 该算法与主分片数相关，这也是**分片数一旦确定后便不能更改**的原因

### 文档创建的流程

1. Client向node3发起创建文档的请求

2. node3通过routing计算该文档应该存储在Shard1上，查询cluster state后确认主分片P1在node2上，然后转发创建文档的请求到node2

3. P1接收并执行创建文档请求后，将同样的请求发送到副本分片R1

4. R1接收并执行创建文档请求后，通知P1成功的结果

5. P1接收副本分片结果后，通知node3创建成功

6. node3返回结果到Client

![文档创建流程](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180927134612.png)

### 文档读取的流程

1. Client向node3发起获取文档1的请求
2. node3通过routing计算该文档在Shard1上，查询cluster state后获取Shard1的主副分片列表，然后以轮询的机制获取一个shard，比如这里是R1，然后转发读取文档的请求到node1
3. R1接收并执行读取文档请求后，将结果返回node3
4. node4返回结果给Client

![文档读取的流程](https://github.com/EmonCodingBackEnd/ElasticStack/blob/master/Elasticsearch/src/main/resources/images/20180927135207.png)



# 六、深入了解Search的运行机制

# 七、聚合分析入门

# 八、数据建模

# 九、集群调优建议

