# Elasticsearch

[TOC]

# 一、常见术语

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

### 4.1、es有专门的Index API，用于创建、更新、删除索引配置等

- 创建索引api如下：

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

- 查看现有索引

```
GET _cat/indices
```

- 删除索引

```
request:
DELETE /test_index
response:
{
  "acknowledged": true
}
```



