### 对象和Nested对象

数据存储的时候，内部对象的边界并没有考虑在内，JSON格式被处理成扁平式的键值对结构，因此会产生以下错误的查询结果

```json
DELETE movies
PUT movies/
{
  "mappings": {
    "properties" : {
        "actors" : {
          "properties" : {
            "first_name" : {
              "type" : "keyword"
            },
            "last_name" : {
              "type" : "keyword"
            }
          }
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
  }
}
POST movies/_doc/1
{
	"title": "Speed",
	"actors": [
		{"first_name": "Xu", "last_name":"Junyi"},
		{"first_name": "Feng", "last_name": "Wenxing"}
	]
}

//竟然能够搜索到结果
//存储的时候是这样的
//{"first_name": ["Xu", "Feng"]}
//{"last_name": ["Junyi"， "Wenxing"]}
POST movies/_search
{
  "query": {
    "bool": {
      "must": [
        {"match":{"actors.first_name": "Xu"}},
        {"match":{"actors.last_name": "Wenxing"}}
        ]
    }
  }
}

//解决办法:使用Nested Data Type解决
DELETE movies
PUT movies/
{
  "mappings": {
    "properties" : {
        "actors" : {
          "type": "nested", 
          "properties" : {
            "first_name" : {
              "type" : "keyword"
            },
            "last_name" : {
              "type" : "keyword"
            }
          }
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
      }
  }
}
POST movies/_doc/1
{
	"title": "Speed",
	"actors": [
		{"first_name": "Xu", "last_name":"Junyi"},
		{"first_name": "Feng", "last_name": "Wenxing"}
	]
}
//Nested查询
POST movies/_search
{
	"query": {
		"bool": {
			"must": [
				{"match":{"title": "Speed"}},
				{"nested": {
					"path": "actors",
					"query": {
						"bool": {
							"must": [
								{"match": {"actors.first_name": "Xu"}},
								{"match": {"actors.last_name": "Wenxing"}}
							]
						}
					}
				}}
			]
		}
	}
}
POST movies/_search
{
	"query": {
		"bool": {
			"must": [
				{"match":{"title": "Speed"}},
				{"nested": {
					"path": "actors",
					"query": {
						"bool": {
							"must": [
								{"match": {"actors.first_name": "Xu"}},
								{"match": {"actors.last_name": "Junyi"}}
							]
						}
					}
				}}
			]
		}
	}
}
// Nested Aggregation
// 普通的Aggregation是不工作的
POST movies/_search
{
	"size": 0,
	"aggs": {
		"actors_aggs": {
			"nested": {
				"path": "actors"
			},
			"aggs": {
				"actor_name": {
					"terms": {
						"field": "actors.first_name"
					}
				}
			}
		}
	}
}
```



### 父子文档

- 对象和Nested对象的局限性：在每次更新文档的时候，需要重新索引整个对象
- 可以通过维护Parent/Child的关系，从而分离两个对象；父文档和子文档是两个独立的文档
- 父文档和子文档必须存在相同的分片上，确保查询join的性能，通过指定routing确保

```json
//定义索引
PUT my_blogs
{
	"settings": {
		"number_of_shards": 2
	},
	"mappings": {
		"properties": {
			"title": {
				"type": "keyword"
			},
			"context": {
				"type": "text"
			},
			"blog_comment_relation": {
				"type": "join",
				"relations": {
					"blog": "comment"
				}
			}
		}
	}
}
//索引父文档
PUT my_blogs/_doc/blog1
{
	"title": "Learning Elasticsearch",
	"content": "learning ELK @ geektime",
	"blog_comment_relation": {
		"name": "blog"
	}
}
PUT my_blogs/_doc/blog2
{
	"title": "Learn Java",
	"content": "learning Spring",
	"blog_comment_relation": {
		"name": "blog"
	}
}

//索引子文档
PUT my_blogs/_doc/comment1?routing=blog1
{
	"username": "Zhao",
	"comment": "I don't like Elasticsearch",
	"time": "2020年2月11日09:30:02",
	"blog_comment_relation": {
		"name": "comment",
		"parent": "blog1"
	}
}
PUT my_blogs/_doc/comment2?routing=blog2
{
	"username": "Qian",
	"comment": "I like Java Programming",
	"time": "2020年2月16日09:30:02",
	"blog_comment_relation": {
		"name": "comment",
		"parent": "blog2"
	}
}

//查询所有文档
POST my_blogs/_search

//根据父文档ID查询，不显示子文档的信息
GET my_blogs/_doc/blog1

//通过父文档ID查询评论内容
POST my_blogs/_search
{
	"query": {
		"parent_id": {
			"type": "comment",
			"id": "blog2"
		}
	}
}

//通过父文档的信息查找子文档
POST my_blogs/_search
{
	"query": {
		"has_parent": {
			"parent_type": "blog",
			"query": {
				"match": {
					"title": "Learn Java"
				}
			}
		}
	}
}
//通过子文档的信息查找父文档
POST my_blogs/_search
{
	"query": {
		"has_child": {
			"type": "comment",
			"query": {
				"match": {
					"username": "Qian"
				}
			}
		}
	}
}

//通过子文档ID访问是不行的，似乎又是可行的
GET my_blogs/_doc/comment1

//需要指定routing
GET my_blogs/_doc/comment1?routing=blog1

//更新子文档
PUT my_blogs/_doc/comment1?routing=blog1
{
	"username": "Zhao",
	"comment": "I like Elasticsearch",
	"time": "2020年2月12日09:30:02",
	"blog_comment_relation": {
		"name": "comment",
		"parent": "blog1"
	}
}
```



### Nested对象 VS 父子文档

|          | Nested Object                        | Parent/Child                       |
| :------: | ------------------------------------ | ---------------------------------- |
|   优点   | 文档存储在一起，读取性能高           | 父子文档可以独立更新               |
|   缺点   | 更新嵌套的子文档时，需要更新整个文档 | 需要额外的内存维护关系，读取性能差 |
| 使用场景 | 子文档偶尔更新，以查询为主           | 子文档更新频繁                     |

### 重建索引

什么时候需要重建索引：

- Mappings修改：字段类型改变，分词器及字典更新
- Settings修改：修改主分片数
- 集群内，集群间做数据迁移

Elasticsearch内置的API：

- reindex：在其他索引上重建索引
- update by query：在现有索引上重建

#### update by query

```json
//为索引增加子字段
DELETE blogs
PUT blogs/_doc/1
{
	"content": "Hadoop is wonderful",
	"keyword": "Hadoop"
}

GET blogs/_mapping

//修改mapping，增加一个子字段
PUT blogs/_mapping
{
	"properties": {
		"content": {
			"type": "text",
			"fields": {
				"english": {
					"type": "text",
					"analyzer": "english"
				}
			}
		}
	}
}
//修改后再增加一个文档，按理说其没有子字段
PUT blogs/_doc/2
{
	"content": "Java is Good",
	"keyword": "Java"
}
//查询新写入的文档
POST blogs/_search
{
	"query": {
		"match": {
			"content.english": "Java"
		}
	}
}
//查询修改前的文档，结果为无
POST blogs/_search
{
	"query": {
		"match": {
			"content.english": "Hadoop"
		}
	}
}
//重建索引
POST blogs/_update_by_query
{}

//查询修改前的文档，有结果了
POST blogs/_search
{
	"query": {
		"match": {
			"content.english": "Hadoop"
		}
	}
}
```

#### reindex

- 修改索引的主分片数
- 修改Mapping字段的类型
- 集群内、集群建数据迁移

```json
//修改字段的类型

GET blogs/_mapping
//会报错
PUT blogs/_mapping
{
	"properties" : {
        "content" : {
          "type" : "text",
          "fields" : {
            "english" : {
              "type" : "text",
              "analyzer" : "english"
            }
          }
        },
        "keyword" : {
          "type" : "keyword"
        }
      }
}

//创建新的索引并且需要设置好Mapping
PUT blogs_fix/
{
	"mappings": {
		"properties" : {
	        "content" : {
	          "type" : "text",
	          "fields" : {
	            "english" : {
	              "type" : "text",
	              "analyzer" : "english"
	            }
	          }
	        },
	        "keyword" : {
	          "type" : "keyword"
	        }
	    }
	}
}

//重建索引
POST _reindex
{
	"source": {
		"index": "blogs"
	},
	"dest": {
		"index": "blogs_fix"
	}
}

GET blogs_fix/_doc/1

//测试Term Aggregation
POST blogs_fix/_search
{
  "size": 0,
	"aggs": {
		"blog_keyword": {
			"terms": {
				"field": "keyword"
			}
		}
	}
}

```

```json
//reindex只会创建不存在的文档，如果文档已经存在，会导致版本的冲突，可以添加op_type，当文档存在的时候不执行
POST _reindex
{
	"source": {
		"index": "user"
	},
	"dest": {
		"index": "user2",
		"op_type": "create"
	}
}

//异步操作
POST _reindex?wait_for_completion=false
GET _tasks?detailed=true&actions=*reindex
```

### Ingest Pipeline

修复和增强写入的数据：为某个字段设置默认值，重命名字段的名字，对字段值进行split操作

```json
PUT users/_doc/1
{
	"name": "Xu junyi",
	"interests": "reading, writing, programming",
	"gender": "male"
}
//测试
POST _ingest/pipeline/_simulate
{
	"pipeline": {
		"description": "users pipeline",
		"processors": [
			{"split": {"field":"interests", "separator": ", "}},
			{"set": {"field": "views", "value": 0}}
		]
	},
	"docs": [
		{
			"_source": {
				"name": "Zhao Xiaoming",
				"interests": "reading, writing, programming",
				"gender": "male"
			}
		},
		{
			"_source": {
				"name": "Wang Quansheng",
				"interests": "musicing, painting, programming",
				"gender": "female"
			}
		}
	]
}
//创建一个pipeline
PUT _ingest/pipeline/user_pipeline
{
	"description": "users pipeline",
	"processors": [
		{"split": {"field":"interests", "separator": ", "}},
		{"set": {"field": "views", "value": 0}}
	]
}
//查看pipeline
GET _ingest/pipeline/user_pipeline

//测试创建的pipeline
POST _ingest/pipeline/user_pipeline/_simulate
{
	"docs": [
		{
			"_source": {
				"name": "Zhao Xiaoming",
				"interests": "reading, writing, programming",
				"gender": "male"
			}
		},
		{
			"_source": {
				"name": "Wang Quansheng",
				"interests": "musicing, painting, programming",
				"gender": "female"
			}
		}
	]
}


DELETE users
//不使用pipeline创建文档
PUT users/_doc/1
{
	"name": "Xu junyi",
	"interests": "reading, writing, programming",
	"gender": "male"
}
//使用pipeline创建文档
PUT users/_doc/2?pipeline=user_pipeline
{
	"name": "Wang Quansheng",
	"interests": "musicing, painting, programming",
	"gender": "female"
}
//一条被处理，一条没有被处理
GET users/_search
{}
//执行update_by_query会报错
POST users/_update_by_query?pipeline=user_pipeline

//需要指定query条件
POST users/_update_by_query?pipeline=user_pipeline
{
	"query": {
		"bool": {
			"must_not": {
				"exists": {"field": "views"}
			}
		}
	}
}
//全部被处理了
GET users/_search
{}
```

### Ingest Node VS Logstash

|                |            Logstash            |            Ingest Node             |
| :------------: | :----------------------------: | :--------------------------------: |
| 数据输入和输出 |   多数据源读取，写入多数据源   |   从ES REST API获取数据，写入ES    |
|    数据缓冲    | 实现了简单的数据队列，支持重写 |               不支持               |
|    数据处理    | 支持大量的插件，也支持定制开发 | 内置的插件，可以开发Plugin进行扩展 |
|   配置和使用   |     增加了一定的架构复杂度     |           无需额外的部署           |



### Painless Script

- Painless支持所有Java 的数据类型和Java API子集，可以对文档字段进行加工处理
- 高性能，安全
- 支持显示类型或者动态定义类型
- Elasticsearch会将脚本编译后缓存在Cache中，编译的开销比较大，默认缓存100个脚本

通过Painless脚本访问字段的方式：

- Ingestion：ctx.field_name
- Update：ctx._source.field_name
- Search & Aggregation：doc["field_name"]

```json
//测试，添加一个painless script
POST _ingest/pipeline/_simulate
{
	"pipeline": {
		"description": "users pipeline",
		"processors": [
			{"split": {"field":"interests", "separator": ", "}},
			{"set": {"field": "views", "value": 0}},
			{
				"script": {
					"source": """
						if(ctx.containsKey("name")){
							ctx.name_length = ctx.name.length();
						} else {
							ctx.name_length = 0
						}
					"""
				}
			}
		]
	},
	"docs": [
		{
			"_source": {
				"name": "Zhao Xiaoming",
				"interests": "reading, writing, programming",
				"gender": "male"
			}
		},
		{
			"_source": {
				"name": "Wang Quansheng",
				"interests": "musicing, painting, programming",
				"gender": "female"
			}
		}
	]
}

DELETE users
PUT users/_doc/1
{
	"name": "Xu junyi",
	"interests": "reading, writing, programming",
	"gender": "male",
	"views": 0
}
//在update中使用脚本
POST users/_update/1
{
	"script": {
		"source": "ctx._source.views += params.new_views",
		"params": {
			"new_views": 3
		}
	}
}
GET users/_doc/1

//保存脚本到Cluster State
POST _scripts/update_views
{
	"script": {
		"lang": "painless",
		"source": "ctx._source.views += params.new_views"	
	}
}
//使用保存的脚本
POST users/_update/1
{
	"script": {
		"id": "update_views",
		"params": {
			"new_views": 10
		}
	}
}
GET users/_doc/1

//在search时候使用脚本
GET users/_search
{
	"script_fields": {
		"rnd_views": {
			"script": {
				"lang": "painless",
				"source": """
					java.util.Random rnd = new Random();
					doc['views'].value + rnd.nextInt(10);
				"""
			}
		}
	}
}
```



### 数据建模

数据建模是构建数据模型的过程，需要考虑功能需求和性能需求等

- 实体属性、实体之间的联系、搜索相关的配置
- 分片数量

- 字段配置：字段类型，字段是否被搜索及分词，是否被聚合及排序，是否要额外的存储
- 性能需求

#### 对字段进行建模

需要考虑：字段类型，字段是否被检索及分词，是否被聚合及排序，是否要额外的存储

字段类型：Text VS Keyword
- Text
	- 用于全文搜索，文本会被analyzer分词
	- 默认不支持聚合和排序，需要设置`fielddata`为true
- Keyword
	- 用于id，枚举以及不需要分词的文本。例如电话号码，邮编，地址，性别等
	- 适用于filter（精确匹配），sorting和aggregation
	- 对于数字，也应该设置成keyword，从而能够获得更好的性能
- 设置多字段类型
	- 默认会为文本类型设置成text，并且添加一个keyword的子字段
	- 在处理人类语言的时候，增加english，pinyin或者standard分词器，提高搜索结构

检索

- 如不需要检索，将index设置为false。`"cover_url":{"type":"keyword", "index": false}`，但是仍然是支持聚合分析的

聚合和排序

- 如果不需要聚合和排序，将doc_values / fielddata设置为false
- 对于更新频繁，聚合查询频繁的keyword类型的字段，设置`eager_global_ordinals:true`，能够更好利用缓存的特性，增加term aggregation的性能

额外的存储
- 是否需要专门存储字段的原始内容：将store设置为true，一般结合_source的enabled为false时候使用
- disable `_source`：节省磁盘空间，适用于指标性数据（一般比较少修改）。不过一般先考虑压缩比，因为当关闭之后，无法看到`_source`，也无法做reindex，无法做update

```json
//对于图书来说，图书的内容很多，会导致_source的内容过大，此时可以关闭_source，将每个字段的“store”设置为true
PUT books
{
	"mappings": {
		"_source": {"enabled": false},
		"properties": {
			"author": {"type": "keyword", "store": true},
			"cover_url": {"type": "keyword", "index": false, "store": true},
			"description": {"type": "text", "store": true},
			"content": {"type": "text", "store": true},
			"public_date": {"type": "date", "store": true},
			"title": {
				"type": "text",
				"fields": {
					"keyword": {
						"type": "keyword",
						"ignore_above": 100
					}
				},
				"store": true
			}
		}
	}
}
//查询结果中source不包含数据
POST books/_search
{}

//对于需要显示的内容，指定store_fields
//高亮显示content中匹配的相关信息
POST books/_search
{
	"stored_fields": ["title", "author", "public_date"],
	"query": {
		"match": {
			"content": "searching"
		}
	},
	"highlight": {
		"fields": {
			"content": {}
		}
	}
}
```

### 数据建模的实践

1. 处理关联关系

优先考虑Denormalization；当数据包含多值对象，同时有查询需求的时候，使用Nested；当关联文档更新很频繁的时候使用Parent/Child

2. 避免过多字段

- 过多的字段不容易维护；Mapping信息保存在cluster state中，数据量过大，对集群的性能有影响（cluster state信息需要和所有的节点同步）；删除或者修改数据需要reindex
- 什么时候会产生很多的字段？当打开Dynamic的时候，未知的字段会被自动加入



可以使用Nested Object & Key Value

- 可以减少字段数量，解决cluster state保存过多meta信息的问题
- 但是也导致了查询语句复杂度增加；Nested对象不利于在kibana中实现可视化分析

```json
//对于这些字段，可以使用Nested减少字段数
"cookies": {
	"properties": {
		"age": {"type": "long"},
		"login": {"type": "date"},
		"email": {
			"type": "text",
			"fields": {
				"keyword": {
					"type": "keyword",
					"ignore_above": 256
				}
			}
		},
		"username": {
			"type": "text",
			"fields": {
				"keyword": {
					"type": "keyword",
					"ignore_above": 256
				}
			}
		}
	}
}
//修改成这样
"cookies": {
	"type": "nested",
	"properties": {
		"name": {"type": "keyword"},
		"dateValue": {"type": "date"},
		"keywordValue": {"type": "keyword"},
		"intValue": {"type": "integer"}
	}
}

//这样写入数据
PUT cookie_service/_doc/1
{
	"url": "www.google.com",
	"cookies": [
		{"name": "username", "keywordValue": "tom"},
		{"name": "age", "intValue": 32}
	]
}
PUT cookie_service/_doc/2
{
	"url": "www.baidu.com",
	"cookies": [
		{"name": "login", "dateValue": "2019-01-11"},
		{"name": "email", "keywordValue": "abc@qq.com"}
	]
}
//nested查询比较复杂
POST cookie_service/_search
{
	"query": {
		"nested": {
			"path": "cookies",
			"query": {
				"bool": {
					"filter": [
						{"term": {"cookies.name": "age"}},
						{"range": {
							"cookies.intValue": {"gte": 30}
						}}
					]
				}
			}
		}
	}
}
```



3. 避免正则查询

- 正则，通配符查询，前缀查询都属于Term查询，性能不够好

```json
案例：一个索引保存了Elasticsearch的版本信息，比如“version 7.1.0”，现在要搜索7.x.x的所有版本
//比较好的方法是将字符串转换为对象
"properties": {
	"version": {
		"properties": {
			"display_name": {"type": "keyword"},
			"hot_fix": {"type": "byte"},
			"minor": {"type": "byte"},
			"marjor": {"type": "byte"}
		}
	}
}

PUT ESV/_doc/1
{
	"version": {
		"display_name":"7.1.0",
		"marjor": 7,
		"minor": 1,
		"hot_fix": 0
	}
}
PUT ESV/_doc/2
{
	"version": {
		"display_name":"7.2.1",
		"marjor": 7,
		"minor": 2,
		"hot_fix": 1
	}
}

//查询
POST ESV/_search
{
	"query": {
		"bool": {
			"filter": [
				{"match": {"version.marjor": 7}},
				{"match": {"version.minor": 1}}
			]
		}
	}
}
```



4. 避免空值引起的聚合不准确

```json
PUT ratings/_doc/1
{
	"score": 10
}
PUT ratings/_doc/2
{
	"score": null
}
//聚合的结果不正确
//设置null_value后可以正确计算
POST ratings/_search
{
  "size": 0,
	"aggs": {
		"avg_score": {
			"avg": {
				"field": "score"
			}
		}
	}
}
//解决办法，设置null_value
DELETE ratings
PUT ratings
{
	"mappings": {
		"properties": {
			"score": {
				"type": "integer",
				"null_value": 1
			}
		}
	}
}

```

4. 为索引的Mapping加入meta信息

Mappings的设置是一个迭代的过程，并且很重要，最好能够对Mappings加入meta信息，更好的进行版本管理，可以考虑将Mappings文件上传到git进行管理