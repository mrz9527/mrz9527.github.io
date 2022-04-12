elasticsearch搜索引擎

# 0. 概念和语法

在elasticsearch中。

index：索引，类似于数据库的一张表

mapping：索引的映射，类似于表的表头，包含了表的所有字段以及字段的类型

properties:映射属性，里面烦恼歌如索引的映射的所有字段

text：映射中字段的类型，text表示全文索引，会分词

keyword：映射中的字段类型，keyword表示关键字，不会分词，作为一个整体来搜索

type：在映射中，代表字段的类型，在文档中，代表文档的类型，在分词器中，代表分词器类型



**elasticsearch提供的默认关键字**

**uri关键字**

_open, _close, _cat, _update, _mapping, _doc, _cluster, _analyze

/shop/_open:打开索引

/shop/_close:关闭索引

/_cat/indices?v:查看所有索引

/shop/\_doc:索引名为shop下的文档类型，7.2版本默认只有一个文档类型_doc，不存在多个类型

/shop/_update/1:更该文档内容后更新

/_cluster/settings:集群设置

/_analyze:分词工具，（analyzer是分词器，两者有区别，在实际配置分词器时，用的是analyzer）

**响应或请求中数据的关键字**

 _id, _source, _version, _shards, _score, _source



elasticsearch服务器的uri结构，elasticsearch系统uri以下划线开头，比如_open, _close, _cat, _update, _mapping, _doc等

通用情况下，uri的1级路径是索引名(如shop)，2级路径是文档_doc，3级路径是文档id

localhost:9200/shop/_doc/1

shop:就是索引

1：就是索引中文档id为1的文档



集群设置，自动增加索引

http://localhost:9200/_cluster/settings

auto_create_index

# 1.下载安装、启动、配置文件

官网：elastic.co下载

解压之后，假定es_home代表安装目录

$(es_home)/bin/elasticsearch是可执行文件，直接启动

启动后，默认端口是9200。

$(es_home)/config/elasticsearch.yml是配置文件，可以修改默认端口

浏览器中输入http://localhost:9200，即可成功请求elasticsearch服务器，我这里使用postman工具

# 2. 索引：增删改查

elasticsearch的索引，相当于数据库的一张表

## 创建索引

PUT	localhost:9200/**shop**

这里shop就是我创建的索引，索引在服务器根目录下

PUT	localhost:9200/**nba**

这里nba也是我创建的索引，索引在服务器根目录下

## **查看指定索引**

GET	localhost:9200/**shop**

GET	localhost:9200/**shop,nba**

## **查看所有索引**

GET localhost:9200/_all

或者

GET localhost:9200/_cat/indices?v

## 关闭索引

POST localhost:9200/shop/_close

## 打开索引

POST localhost:9200/shop/_open

# 3.映射mapping

索引的映射，相当于表的表结构（或者表头），也就是字段

## 创建索引的映射

以json的形式创建字段

PUT 	localhost:9200/shop/_mapping -H 'Content-Type:

application/json' -d'

{

​    "properties":{

​        "name":{

​            "type":"text"

​        },

​        "price":{

​            "type":"keyword"

​        },

​        "tag":{

​            "type":"keyword"

​        }

​    }

}

'

![image-20211111162918875](/Users/xm210408/Library/Application Support/typora-user-images/image-20211111162918875.png)

## 查看索引的映射

GET 	localhost:9200/shop/_mapping

GET 	localhost:9200/shop,nba/_mapping

GET 	localhost:9200/shop/_mappings

GET 	localhost:9200/shop,nba/_mappings

## 查看所有索引的映射

GET localhost:9200/_mapping

或者

GET localhost:9200/_all/\_mapping

## 增加映射的字段（不能修改字段的类型）

PUT 	localhost:9200/shop/_mapping -H 'Content-Type:

application/json' -d '

{

​    "properties":{

​        "name":{

​            "type":"text"

​        },

​        "price":{

​            "type":"keyword"

​        },

​        "tag":{

​            "type":"keyword"

​        },

​		**"date":{**

​		**"type":"text"**

​		**}**

​    }

}

'

在原来基础上增加date字段

## 创建索引的同时创建映射和其他配置项，比如分词器

POST	localhost:9200/kikaindex -H 'Content-Type:

application/json' -d '

{

​    "settings":{

​        "analysis":{

​            "analyzer":{

​                "ws_analyze":{

​                    "type":"whitespace"

​                },

​                "s_analyze":{

​                    "type":"standard"

​                }

​             }

​         }

​    },



​    "mappings": {

​        "properties":{

​            "name":{

​                "type":"keyword"

​            },

​            "tag":{

​                "type":"text",

​                "analyzer":"ws_analyze"

​            },

​            "position":{

​                "type":"text",

​                "analyzer":"s_analyze"

​            },

​            "personCount":{

​                "type":"long"

​            }

​        }

​    }

}

'

创建索引kikaime的同时，指定了settings和mappings，

# 4. 文档

索引中的文档，相当于数据库中表的记录record。

## 新增文档

创建指定文档和文档的索引

PUT 	localhost:9200/shop/_doc/1 -H 'Content-type:

application/json' -d '

{

​	"name":"小米",

​	"price":"1.2",

​	"tag":"属于食品",

​	"date":"2021.9.10"

}

'

![image-20211111172020265](/Users/xm210408/Library/Application Support/typora-user-images/image-20211111172020265.png)

创建文档时不指定文档内的id，会自动创建文档索引

![image-20211111172304123](/Users/xm210408/Library/Application Support/typora-user-images/image-20211111172304123.png)

## 查看文档

![image-20211111171928445](/Users/xm210408/Library/Application Support/typora-user-images/image-20211111171928445.png)

## 查看多个文档

post	localhost:9200/_mget -H 'Content-Type:

application/json' -d '

{

​		"docs":[

​				{

​						"_index":"shop",

​						"_type":"\_doc",

​						"_id":"1"

​				}

​		]

}

'

或者在指定索引下查看多个文档

POST	localhost:9200/shop/_mget -H 'Content-Type:

application/json' -d '

{

​		"docs":[

​				{

​						"_type":"\_doc",

​						"_id":"1"

​				}

​		]

}

'

## 修改文档

POST	localhost:9200/shop/_update/1 -H 'Content-Type:

application/json' -d '

{

​	"name":"小米",

​	**"price":"2",**

​	"tag":"属于食品",

​	"date":"2021.9.10"

}

'

# 5. 搜索

term：词条查询，词条查询不会分析查询条件，

full text：全文查询

## term查询

POST	localhost:9200/shop/_search 	-H 'Content-Type:

application/json'	-d '

{

​	"query":{

​		"term":{

​				"name":"来一桶"

​			}

​	}

}

'

term查询不支持一个字段查询多个值，比如只能查“来一桶”，不能一次查“来一桶”和“小米”。

## full text查询

### match_all查询全部

match_all默认只显示10条，如果想显示全部，可以指定分页

POST	localhost:9200/shop/_search 	-H 'Content-Type:

application/json'	-d '

{

​	"query":{

​		"match_all":{}

​	},

​	"from":0

​	"size":100

}

'

这里从0开始，显示100条结果



### match普通查询

POST	localhost:9200/shop/_search 	-H 'Content-Type:

application/json'	-d '

{

​	"query":{

​		"match":{

​			"name":"小米"

​		}

​	}

}

'

### multi_match多字段查询

POST	localhost:9200/nba/_search 	-H 'Content-Type:

application/json'	-d '

{

​    "query":{

​        "multi_match":{

​            "query":"科比",

​            "fields":["team","name"]

​        }

​    }

}

'

### match_phrase

类似term词条查询，查询字符串不会拆分

语法类似match和term

### match_phrase_prefix 前缀查询

match_phrase_prefix语法和match和term一致，一般针对英文单词来进行前缀查询

# 6. 分词

查看分词效果

POST	localhost:9200/_analyze 	-H 'Content-Type:

application/json'	-d '

{

​    "analyzer":"standard",

​    "text":"best 3-points Curry!"

}

'

![image-20211112103331923](/Users/xm210408/Library/Application Support/typora-user-images/image-20211112103331923.png)

## 常见分词器

## 内置分词器analyzer

​	standard:标准分词器是默认的分词器，大写会变成小写，字母和数字以外的字符会丢弃

​	simple：simple分词器，大写变成小写，字母以外的字符会丢弃

​	whitespace：空白字符分词器，按照空白符分词，所有字符都不会丢弃（除了空白字符）

​	stop：停用词分词器，在simple基础上，将停用词丢弃（the is at in of等等会被丢弃），会有一个专门的停用词列表

​	language：特定语言的分词器，比如英语english，内置语言：几十种海外语言

​	pattern：正则分词器，使用正则表达式将文本进行分词，默认正则表达式是\W+，（这种默认表达式，保留字母和数字，其余字符丢弃），可以修改正则表达式。



## 中文分词器

内置分词器对中文不友好，需要使用中文分词器。

### smartcn分词器

是中文或中英混合分词器

安装：进入$(es_home)/bin/目录，在该目录下执行

```sh
sh elasticsearch-plugin install analysis-smartcn
```

安装完成后重启es服务，即可使用该分词器。

重新启动es时可能会报”org.elasticsearch.bootstrap.StartupException: java.lang.IllegalStateException: failed to obtain node locks“错误，是因为上一次启动的es没有关闭，需要关闭。

ps -ef | grep 'elastic'查看es进程号，然后kill关闭该进程。然后再启动。



### ik分词器

下载：https://github.com/medcl/elasticsearch-analysis-ik/releases

下载与es版本一致的ik版本，本地使用的es是7.2.1，所以ik也选择7.2.1

ik分词器的型别为：ik_max_word

![image-20211112144355134](/Users/xm210408/Library/Application Support/typora-user-images/image-20211112144355134.png)

# 7. 常见字段类型

## 字符串类型

text

keyword

## 数值型

long,integer,short,byte,double,float,half_float,scaled_float

## 布尔型

boolean

## 二进制

binary

该类型的字段把值当做经过base64编码的字符串，默认不存储，且不可搜索

## 范围类型

范围类型表示值是一个范围，而不是一个具体值

integer_range, float_range, long_range, double_range, date_range

比如age的类型是integer_range，那么值可以是{"gte":20, "lte":40}; 搜索"term":{"age":21}可以搜索该值

gte:大于等于，greate than equal

lte:小于等于

## 日期类型

date

## 数组类型

## 对象类型

## 专有类型

ip地址
