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
GET /test_index/_mapping
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









# 四、Search API介绍

# 五、分布式特性介绍

# 六、深入了解Search的运行机制

# 七、聚合分析入门

# 八、数据建模

# 九、集群调优建议

