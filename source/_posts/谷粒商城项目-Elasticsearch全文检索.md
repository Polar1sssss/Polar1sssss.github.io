---
title: 谷粒商城项目-Elasticsearch全文检索
date: 2021-07-14 14:10:05
tags:
 - Java
 - 项目开发
---

# 基本概念

ElasticSearch可以快速地存储、搜索和分析海量数据。底层是Lucene，ElasticSearch对Lucene进行了封装，提供了REST API（天然跨平台）接口，开箱即用。

## Index（索引）

动词，相当于MySQL中的insert；

名词，相当于MySQL中的Database

## Type（类型）

在Index中，可以定义一个或多个Type；

类似于MySQL中的Table，每一种类型的数据放在一起

## Document（文档）

保存在某个索引（Index）下，某种类型（Type）的一个数据（Document），文档是JSON格式，Document就像是MySQL中的某个Table里面的内容。

## 倒排索引机制

**分词**：将整句拆分成单词。

保存5条数据：

1、红海行动

2、探索红海行动

3、红海特别行动

4、红海纪录片

5、特工红海特别探索

| 词     | 记录          |
| ------ | ------------- |
| 红海   | 1，2，3，4，5 |
| 行动   | 1，2，3       |
| 探索   | 2，5          |
| 特别   | 3，5          |
| 纪录片 | 4             |
| 特工   | 5             |

**检索**：

1）红海特工行动（3号记录命中率2/3，5号记录命中率2/4） 

2）红海行动

# Docker安装

## 下载镜像文件

```bash
docker pull elasticsearch:7.4.2
docker pull kibana -- 可视化检索
```

## 创建实例

### ElasticSearch

1. 创建配置及数据文件夹

    ```bash
    mkdir -p /mydata/elasticsearch/config
    mkdir -p /mydata/elasticsearch/data
    echo "http.host: 0.0.0.0" >> /mydata/elasticsearch/config/elasticsearch.yml
    ```

1. 启动docker

    ```bash
    docker run --name elasticsearch -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms64m -Xmx512m" -v /mydata/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /mydata/elasticsearch/data:/usr/share/elasticsearch/data -v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins -d elasticsearch:7.4.2 
    ```

### Kibana

```bash
docker run --name kibana -e ELASTICSEARCH_URL=http://192.168.146.128:9200 -p 5601:5601 -d kibana:7.4.2
```

# 初步检索

## _cat

GET /_cat/nodes：查看所有节点

GET /_cat/health：查看es健康状态

GET /_cat/master：查看主节点

GET /_cat/indices：查看所有索引 show databases;

## 索引一个文档（保存）

保存一个数据，保存在哪个索引的哪个类型下，指定用那个唯一标识

PUT customer/external/1：必须带id，在customer索引的external类型下保存1号数据，再次发送请求就是更新操作

POST customer/external/：可以不带id，新增一条数据，自动生成id，再次发送请求重新生成新的id

```json
{
    "name":"Jhon Doe"
}
```

## 查询文档

GET customer/external/1

结果：

```json
{
    "_index": "customer", // 索引
    "_type": "external", // 类型
    "_id": "1", // 唯一标识
    "_version": 1, // 版本号
    "_seq_no": 0, // 并发控制字段，每次更新+1，用来做乐观锁
    "_primary_term": 1, // 同上，主分片重新分配，如重启，就会变化
    "found": true,
    "_source": {
        "name": "Jhon Doe"
    }
}
```

更新时携带?if_seq_no=0&if_primary_term=1：只有满足条件才会更新成功

## 更新文档

POST customer/external/1/_update：带update更新数据中必须加“doc”，如更新数据与原数据对比无变化，则不会进行更新操作（noop无操作）

```json
{
    "doc":{
        "name":"Jhon Abc"
    }
}
```

POST customer/external/1

```json
{
    "name":"Jhon Abc"
}
```

PUT customer/external/1

```json
{
    "name":"Jhon Abc"
}
```

## 删除文档&索引

DELETE customer/external/1

DELETE customer

## bulk批量API

POST customer/external/_bulk

```json
{"index":{"_id":"1"}} 
{"name":"zhangsan"}

{"index":{"_id":"2"}}
{"name":"lisi"}
```

```json
语法格式：
{action:{metadata}}\n
{request body    }\n
{action:{metadata}}\n
{request body    }\n
```

POST /_bulk

```json
{"delete":{"_index":"website","_type":"blog","_id":"123"}}
{"create":{"_index":"website","_type":"blog","_id":"123"}}
{"title":"My First Blog post"}
{"index":{"_index":"website","_type":"blog"}}
{"title":"My Second Blog post"}
{"update":{"_index":"website","_type":"blog","_id":"123"}}
{"doc":{"title":"My updated blog post"}}
```

# 进阶检索

## Search API

ES支持两种基本方式检索：

- 通过REST request URI，发送搜索参数（uri+检索参数）

  GET bank/_search?q=*&sort=account_number:asc

- 通过REST request body来发送他们（uri+请求体）

  GET bank/_search

  ```json
  {
      "query"：{
      	"match_all":{}
  	},
  	"sort":[
          {
              "account_number":"asc"
          }
      ]
  }
  ```

## Query DSL

### 基本语法格式

ElasticSearch提供了一个可以执行查询JSON风格的DSL（domain-specific language 领域特定语言）。

- 查询语句的典型结构：

    ```json
    {
        QUERY_NAME:{
            ARGUMENT:VALUE,
            ARGUMENT:VALUE
        }
    }
    ```

- 如果是针对某个字段，其结构为：

    ```JSON
    {
        QUERY_NAME:{
            FIELD_NAME:{
                ARGUMENT:VALUE,
                ARGUMENT:VALUE
            }
        }
    }
    ```

### 返回部分字段

GET bank/_search

```json
{
    "query":{
        "match_all":{}
    },
    "from":0,
    "size":5,
    "_source":["age","balance"]
}
```

### match【匹配查询】

- 基本类型（非字符串），精确匹配

  GET bank/_search

  ```json
  {
      "query":{
          "match":{
              "account_number":"20"
          }
      }
  }
  ```

- 字符串，全文检索，对检索条件进行分词匹配

  GET bank/_search

  ```json
  {
      "query":{
          "match":{
              "address":"mill"
          }
      }
  }
  ```

### match_phrase【短语匹配】

将需要匹配的值当成一个整体（不分词）进行匹配。

GET bank/_search

```json
{
    "query":{
        "match_phrase":{
            "address":"mill road"
        }
    }
}
```

查出address中**包含**mill road的所有记录，并给出相关性得分。

GET bank/_search

```json
{
    "query":{
        "match":{
            "address.keyword":"mill road"
        }
    }
}
```

查出address中**等于**mill road的所有记录，并给出相关性得分。

### multi_match【多字段匹配】

GET bank/_search

```json
{
    "query":{
        "multi_match":{
            "query":"mill movico",
            "fields":["state","address"]
        }
    }
}
```

查出state或者address中包含mill或movico的记录，并给出相关性得分。

### bool【复合查询】

- must：必须符合must列举的所有条件

  ```json
  {
      "query":{
          "bool":{
              "must":[
                  {"match":{"address":"mill"}},
                  {"match":{"gender":"M"}}
              ],
              "must_not":[
                  {"match":{"age":40}}
              ]
          }
      }
  }
  ```

- should：应该满足should列举的条件，不满足也可以，满足得分会更高。如果query中只有should且只有一种匹配规则，那么should的条件就会被作为默认匹配条件而去改变查询结果。

### filter【结果过滤】

使用filter过滤出来的结果不会产生相关性得分。

### term

和match一样，匹配某个属性的值。全文检索字段用match，其他非text字段（如数字）匹配用term。

GET bank/_search

```json
{
    "query":{
        "term":{
            "account_number":"20"
        }
    }
}
```

### aggregations【聚合】

聚合提供了从数据中分组和提取数据的能力。最简单的聚合方法大致等于SQL GROUP BY和聚合函数。

- 搜索address中包含mill的所有人的年龄分布及平均年龄，但不显示这些人的详情

  GET bank/_search

  ```json
  {
      "query":{
          "match":{
              "address":"mill"
          }
      },
      "aggs":{
          "group_by_state":{
              "terms":{
                  "field":"age"
              }
          },
          "avg_age": {
            	"avg": {
              	"field": "age"
            	}
          }
      },
     	"size":0
  }
  ```

- 按照年龄聚合，并求出这些年龄段的平均薪资

  ```json
  {
      "query":{
          "match_all":{},
      },
      "aggs":{
          "ageAgg":{
              "terms":{
                  "field":"age"
              },
              "aggs":{
                  "avgBalance":{
                      "avg":{
                          "fields":"balance"
                      }
                  }
              }
          }
      }
  }
  ```

## Mapping

### 创建映射

PUT /my_index

```json
{
    "mappings":{
        "properties":{
            "age":{"type":"integer"},
            "email":{"type":"keyword"},
            "name":{"type":"text"}
        }
    }
}
```

### 添加新的字段映射

PUT /my_index/_mapping

```json
{
    "properties":{
        "employee_id":{
            "type":"keyword",
            "index":false
        }
    }
}
```

### 更新映射

不能更新已存在的映射字段，想要更新必须创建新的索引进行数据迁移。

### 数据迁移

先创建新的索引，并配置好映射，然后使用如下方法进行数据迁移：

POST _reindex 【固定写法，适用于6.0后不用type保存文档的情况】

```json
{
    "source":{
        "index":"bank"
    },
    "dest":{
        "index":"new_bank"
    }
}
```

POST _reindex 【6.0之前数据迁移】

```json
{
    "source":{
        "index":"bank",
        "type":"account"
    },
    "dest":{
        "index":"new_bank"
    }
}
```

## 分词	

一个tokenizer接收一个字符流，将之分隔为独立的tokens（词元，通常是独立的单词），然后输出tokens流。

### 安装ik分词器

**注意**：不能用默认elasticsearch-plugin install xxx.zip进行自动安装

进入es容器内部plugins目录，创建ik文件夹，并把压缩包解压到ik目录下

```bash
docker exec -it 容器id /bin/bash
wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.4.2/elasticsearch-analysis-ik-7.4.2.zip
unzip 下载的文件
rm -rf *.zip
mv elasticsearch/ik
```

检查是否安装好分词器

```json
cd ../bin
elasticsearch-plugin list
```

### 测试分词器

GET my_index/_analyze

```json
{
    "analyzer":"ik_smart",
    "text":"我是中国人"
}
```

### 自定义词库

修改/usr/share/elasticsearch/plugins/ik/config/中的IKAnalyzer.cfg.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
    <comment>IK Analyzer扩展配置</comment>
    <entry key="ext_dict"></entry>
    <extry key="ext_stopwords"></extry>
    <!--用户可以在这里配置远程扩展词典-->
    <entry key="remote_ext_dict">http://192.168.128.130/fenci/myword.txt</entry>
    <!--<entry key="remote_ext_stopwords">words_location</entry>-->
</properties>
```

# Elasticsearch-Rest-Client

1. 9300：TCP
   - spring-data-elasticsearch:transport-api.jar
     - springboot版本不同，transport-api.jar不同，不能适配es版本
     - 7.x不建议使用，8之后废弃
2. 9200：HTTP
   - JestClient：非官方，更新慢
   - RestTemplate：模拟发送HTTP请求，ES很多操作需要自己封装
   - HttpClient：同上
   - Elasticsearch-Rest-Client（elasticsearch-rest-high-level-client）：官方RestClient，封装了ES操作，API层次分明

# 附录：安装Nginx

- 随便启动一个Nginx实例，为了复制配置

  ​	docker run -p 80:80 --name nginx -d nginx:1.10

- 将容器内的配置文件拷贝到当前目录：docker container cp nginx:/etc/nginx .

- 修改文件名称：mv nginx conf，把这个conf移动到/mydata/nginx下

- 终止原容器：docker stop nginx

- 执行命令删除原容器：docker rm  $ContainerId

- 创建新的nginx，执行以下命令：

  ```bash
  docker run -p 80:80 --name nginx -v /mydata/nginx/html:/usr/share/nginx/html -v /mydata/nginx/logs:/var/log/nginx -v /mydata/nginx/conf:/etc/nginx -d nginx:1.10
  ```

  
