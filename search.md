### Search

- URI Search：在URL中使用查询参数
- Request Body Search：使用Elasticsearch提供的，基于JSON格式的更加完备的Query Domain Specific Language

查询集群上所有的内容：`/_search`

指定查询以index开头的索引：`/index*/_search`

指定查询index1和index2索引的内容：`/index1,index2/_search`

#### 衡量相关性

- Precision（查准率）：尽量返回较少的不相关文档
- Recall（查全率）：尽量返回较多的相关文档
- Ranking：是否能够按照相关度进行排序

#### URI Search

```http
//查找title字段为2012的文档
GET /movies/_search?q=title:2012
{
	"profile":"true"
}
//Phrase Query，title字段包括Beautiful Mind，顺序一致，中间不能有其他单词
GET /movies/_search?q=title:"Beautiful Mind"
{
	"profile":"true"
}
//Term Query，title字段包括Beautiful或者Mind
GET /movies/_search?q=title:(Beautiful Mind)
{
	"profile":"true"
}
//布尔操作，AND / OR / NOT 或者 && / || / !
GET /movies/_search?q=title:(Beautiful NOT Mind)
{
	"profile":"true"
}
//分组操作，+表示must，-表示must_not
//必须要有value1，不要有value2
GET /movies/_search?q=title:(+value1 -value2)
{
	"profile":"true"
}
//范围查询， []闭区间，{}开区间
year:{2019 TO 2020}
year:{* TO 2020}
//算数符号
year:>2010
year:(>2010 && <=2019)
//模糊匹配和近似度匹配
//可以查到beautiful这个单词
GET /movies/_search?q=title:beautifl~1
{
	"profile":"true"
}
//可以查到Lord of the Rings
GET /movies/_search?q=title:"Lord Rings"~2
{
	"profile":"true"
}
//通配符、正则
```

#### Request Body Search

```http
//显示指定的字段
//实现类似分页的效果
POST /kibana_sample_data_ecommerce/_search
{
	"_source":["order_date","category.keyword"],
	"sort":[{"order_date":"desc"}],
	"from":10,	
	"size":20,
	"query":{
		"match_all":{}
	}
}
//脚本字段
POST /kibana_sample_data_ecommerce/_search
{
	"script_fields": {
		"new_field": {
			"script": {
				"lang":"painless",
				"source":"doc['order_date'].value+'_hello'"
			}
		}
	},
	"query": {
		"match_all":{}
	}
}
//查找Last OR Christmas
POST movies/_search
{
	"query": {
		"match": {
			"title":"Last Christmas"
		}
	}
}
//查找Last AND Christmas
POST movies/_search
{
	"query": {
		"match": {
			"title":"Last Christmas",
			"operator":"and"
		}
	}
}
//phrase query,one AND love，且顺序一致
POST movies/_search
{
	"query": {
		"match_phrase": {
			"title" :{
				"query":"one love"
			}
		}
	}
}
//phrase query，中间可以有1个模糊单词
POST movies/_search
{
	"query": {
		"match_phrase": {
			"title" :{
				"query":"one love",
				"slop":1
			}
		}
	}
}
```

##### Query String

```http
PUT /users/_doc/1
{
	"name":"Xu junyi",
	"about":"java, python, elasticsearch"
}
PUT /users/_doc/2
{
	"name":"Feng junyi",
	"about":"C++"
}

POST users/_search
{
	"query": {
		"query_string":{
			"default_field":"name",
			"query":"Xu AND junyi"
		}
	}
}

POST users/_search
{
	"query": {
		"query_string":{
			"fields":["name","about"],
			"query":"(Xu AND junyi) OR (Java AND Elasticsearch)"
		}
	}
}
```

##### Simple Query String

- 不支持AND OR NOT，会当做普通字符
- Term之间默认是OR，可以用defaut_operator指定AND
- 支持+代替AND，|代替OR，-代替NOT

```http
POST users/_search
{
    "query": {
    	"simple_query_string": {
            "query":"Xu junyi",
            "fields":["name"],
            "default_operator": "AND"
         }
    }
}

POST users/_search
{
	"query": {
		"simple_query_string": {
			"query":"junyi -Xu",
			"fields":["name"],
			"default_operator": "AND"
		}
	}
}
```



### 基于Term的查询

在Elasticsearch中，Term查询，对输入不做分词。会将输入作为一个整体，在倒排索引中查找准确的词项，并且使用相关度算分公式为每个包含该词项的文档进行相关度算分

```json
POST /products/_bulk
{"index":{"_id":1}}
{"productID":"XHDK-A-1234-#fJ3", "desc":"iPhone"}
{"index":{"_id":2}}
{"productID":"KDKE-B-1234-#kL5", "desc":"iPad"}


//在文档被索引后，字段会默认进行standard分词
POST /products/_search
{
	"query": {
		"term": {
			"desc": {
				//"value":"iPhone"
				"value":"iphone"
			}
		}
	}
}

POST /products/_search
{
	"query": {
		"term": {
			"productID": {
				//"value":"XHDK-A-1234-#fJ3"
				//"value": "xhdk"
				"value" : "fj3"
			}
		}
	}
}

POST /_analyze
{
  "analyzer": "standard",
  "text": ["XHDK-A-1234-#fJ3"]
}

POST /products/_search
{
	"query": {
		"term": {
			"productID.keyword": {
				"value":"XHDK-A-1234-#fJ3"
			}
		}
	}
}
```



可以使用constant score将查询转换成一个Filtering，避免算分，并且可以使用缓存，提高性能

```json
POST /products/_search
{
	"explain":true,
	"query": {
		"constant_score": {
			"filter": {
				"term": {
					"productID.keyword":"XHDK-A-1234-#fJ3"
				}
			}
		}
	}
}
```



### 结构化查询

```json
DELETE products
POST /products/_bulk
{"index":{"_id":1}}
{"productID":"XHDK-A-1234-#fJ3", "price":10, "avaliable":true, "date":"2010-10-10"}
{"index":{"_id":2}}
{"productID":"KDKE-B-1234-#kL5", "price":20, "avaliable":false, "date":"2020-01-25"}
{"index":{"_id":3}}
{"productID":"KDKE-CKD-1234-#kL5", "price":30, "avaliable":true}

POST /products/_search
{
	"query": {
		"term": {
			"avaliable":true
		}
	}
}
POST /products/_search
{
	"query": {
		"range": {
			"price": {
				"gte":15,
				"lte":40
			}
		}
	}
}
//大于1年前的时间
POST /products/_search
{
	"query": {
		"range": {
			"date": {
				"gte": "now-1y"
			}
		}
	}
}

POST /products/_search
{
	"query": {
		"exists": {
			"field":"date"
		}
	}
}

//处理多值字段
DELETE users
POST /users/_bulk
{"index":{"_id":1}}
{"name":"junyi","interest":"Writing", "count":1}
{"index":{"_id":2}}
{"name":"jiajia","interest":["Writing", "Reading"], "count":2}
//Term查询是包含，而不是等于
//如果只想查找一个值的文档，增加一个genre_count字段进行计数
POST /users/_search
{
	"query": {
		"term": {
			"interest.keyword":"Writing"
		}
	}
}
POST /users/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {
          "interest.keyword": {
            "value": "Writing"
          }
        }},
        {"term": {
          "count": {
            "value": 1
          }
        }}
      ]
    }
  }
}
```



### 相关性算分

Elasticsearch5.0之前是使用TF-IDF，现在是BM25

BM25跟TF-IDF相比，当TF项无限增加时，TF-IDF会不断增加，而BM25会趋于一个数值

#### Boosting Relevance

控制相关度的一种手段，当boot>1，打分的相关度相对性提升，当0<boot<1，相对性降低

```json
PUT testscore/_bulk
{"index":{"_id":1}}
{"content":"we use Elasticsearch to power the search"}
{"index":{"_id":2}}
{"content":"we like elasticsearch"}
{"index":{"_id":3}}
{"content":"you know, for search"}

POST testscore/_search
{
	"explain":true,
	"query": {
		"match": {
			"content":"elasticsearch"
		}
	}
}



//添加boost参数控制相关度
POST testscore/_search
{
	"query": {
		"boosting": {
			"positive": {
				"term": {
					"content": "elasticsearch"
				}
			},
			"negative": {
				"term": {
					"content": "like"
				}
			},
			"negative_boost":0.2
		}
	}
}

POST blogs/_bulk
{"index":{"_id":1}}
{"title":"Apple iPad", "content":"Apple iPad,Apple iPad"}
{"index":{"_id":2}}
{"title":"Apple iPad,Apple iPad", "content":"Apple iPad"}

POST blogs/_search
{
	"query": {
		"bool": {
			"should": [
				{"match": {
					"title": {
						"query":"apple, ipad",
						"boost":1
					}
				}},
				{"match": {
					"content": {
						"query":"apple, ipad",
						//"boost":2
						"boost":0.5
					}
				}}
			]
		}
	}
}
```



### Query Context和Filter Context

- Query Context：会进行相关性算分，must, should
- FilterContext：不会进行算分，会利用缓存，性能更高，filter, must_not

```json
//查询语句的结构会对算分结果产生影响
//同一级下的会具有相同的权重
POST /animals/_search
{
	"query": {
		"bool": {
			"should": [
			{"term":{"text":"brown"}},
			{"term":{"text":"red"}},
			{"bool":{
				"should":[
					{"term":{"text":"quick"}},
					{"term":{"text":"dog"}}
				]
			}}
			]
		}
	}
}
```



### Dis_max Query

单字符串多字段查询

```json
DELETE blogs
PUT /blogs/_bulk
{"index":{"_id":1}}
{"title":"Quick brown rabbits","body":"Brown rabbits are commonly seen."}
{"index":{"_id":2}}
{"title":"Keeping pets healthy", "body":"My quick brown fox eats rabbits on a regular basis."}

//预计是2分数较高，但是结果是1
//算分过程：查询should语句中的两个查询，加和两个查询的评分，乘以匹配语句的总数，除以所有语句的总数
POST /blogs/_search
{
	"explain":true,
	"query": {
		"bool": {
			"should": [
				{"match":{"title":"Brown fox"}},
				{"match":{"body":"Brown fox"}}
			]
		}
	}
}
//不应该将子查询结果简单相加，而是应该找到最佳匹配的评分
//效果和multi_match的best_fields相同
POST /blogs/_search
{
	"explain":true,
	"query": {
		"dis_max": {
			"queries": [
				{"match":{"title":"Brown fox"}},
				{"match":{"body":"Brown fox"}}
			]
		}
	}
}

//结果两个分数一样，但是应该2更高些
POST /blogs/_search
{
	"query": {
		"dis_max": {
			"queries": [
				{"match":{"title":"Quick pets"}},
				{"match":{"body":"Quick pets"}}
			]
		}
	}
}
//添加tie_breaker，会将其他查询结果乘上这个参数，汇总到最终评分上
POST /blogs/_search
{
	"query": {
		"dis_max": {
			"queries": [
				{"match":{"title":"Quick pets"}},
				{"match":{"body":"Quick pets"}}
			],
			"tie_breaker":0.1
		}
	}
}
```

### Multi_match

- best_fields：多个字段之间互相竞争又互相关联，评分来自评分最高的字段
- most_fields：在处理英文内容的时候，字段添加English分词，并且可以加入子字段，设置standard分词，提供更加精确的匹配
- cross_fields：对于地址、人名，需要在多个字段中确认信息，希望能够在列出的字段中找到尽可能多的词

```json
//multi_match默认就是best_fields
POST blogs/_search
{
	"query": {
		"multi_match": {
			"type": "best_fields",
			"query": "Quick pets",
			"fields": ["title", "body"],
			"tie_breaker":0.2,
			"minimun_should_match":"20%"
		}
	}
}

DELETE titles
PUT /titles
{
	"mappings": {
		"properties": {
			"title": {
				"type": "text",
				"analyzer":"english"
			}
		}
	}
}

POST titles/_bulk
{"index":{"_id":1}}
{"title":"My dog bardks"}
{"index":{"_id":2}}
{"title":"I see a lot of barking dogs on the road"}

POST titles/_search
{
	"query": {
		"match": {
			"title":"barking dogs"
		}
	}
}


DELETE titles
PUT /titles
{
	"mappings": {
		"properties": {
			"title": {
				"type": "text",
				"analyzer":"english",
				"fields": {
					"std": {
						"type": "text",
						"analyzer":"standard"
					}
				}
			}
		}
	}
}
GET /titles/_search
{
	"query": {
		"multi_match": {
			"query": "barking dogs",
			"type": "most_fields",
			"fields": ["title", "title.std"]
		}
	}
}



PUT address/_doc/1
{
	"street": "5 Poland Street",
	"city": "London",
	"country": "United Kingdom",
	"postcode": "W1V 3DG"
}

POST address/_search
{
	"query": {
		"multi_match": {
			"query": "Poland Street W1V",
			"type": "most_fields",
			//"operator": "and", 
			"fields": ["street", "city", "country", "postcode"]
		}
	}
}
//query查询的字段都要有
POST address/_search
{
	"query": {
		"multi_match": {
			"query": "Poland Street W1V",
			"type": "cross_fields",
			"operator":"and",
			"fields": ["street", "city", "country", "postcode"]
		}
	}
}
```



### Search Template

```json
POST tmdb/_bulk
{"index":{"_id":1}}
{"title":"space bulk", "overview":"basketball with cartoon aliens"}

POST _scripts/tmdb
{
	"script": {
		"lang": "mustache",
		"source": {
			"_source": ["title", "overview"],
			"size":20,
			"query": {
				"multi_match": {
					"query": "{{q}}",
					"fields": ["title", "overview"]
				}
			}
		}
	}
}
GET _scripts/tmdb
POST tmdb/_search/template
{
	"id":"tmdb",
	"params": {
		"q":"basketball with cartoon aliens"
	}
}
```

### Index Alias

为索引创建别名

```json
PUT movies-2019/_doc/1
{"title":"t1", "overview":"o1", "rating": 10}
POST _aliases
{
	"actions": [
	{
		"add": {
			"index":"movies-2019",
			"alias":"movies-latest",
			"filter": {
				"range": {
					"rating": {
						"gte": 4
					}
				}
			}
		}
	}]
}
POST movies-latest/_search
{
	"query":{"match_all":{}}
}
```



### Function Score Query

```json
DELETE blogs
PUT /blogs/_doc/1
{
	"title": "About popularity",
	"content": "In this post we will talk about...",
	"votes":0
}
PUT /blogs/_doc/2
{
	"title": "About popularity",
	"content": "In this post we will talk about...",
	"votes":100
}
PUT /blogs/_doc/3
{
	"title": "About popularity",
	"content": "In this post we will talk about...",
	"votes":10000
}
//新的算分=老的算分*log(1+votes*factor)
POST /blogs/_search
{
	"query": {
		"function_score": {
			"query": {
				"multi_match": {
					"query":"popularity",
					"fields": ["title", "content"]
				}
			},
			"field_value_factor": {
				"field": "votes",
				"modifier":"log1p",
				"factor":0.1
			}
		}
	}
}

//旧的算分和函数值相加
//multiply, sum, min, max, replace（使用函数值取代算分）
POST /blogs/_search
{
	"query": {
		"function_score": {
			"query": {
				"multi_match": {
					"query":"popularity",
					"fields": ["title", "content"]
				}
			},
			"field_value_factor": {
				"field": "votes",
				"modifier":"log1p",
				"factor":0.1
			},
			"boost_mode":"sum",
			"max_boost":3
		}
	}
}

//让每个用户能够查看不一样的排序结果(广告展现)，但是同一用户看到的结果每次都是一样的
POST /blogs/_search
{
	"query": {
		"function_score": {
			"random_score": {
			  "field":"votes",
				"seed":987
			}
		}
	}
}
```

### Suggester

Suggester就是一种特殊的搜索

#### Term & Phrase

```json
POST articles/_bulk
{"index":{}}
{"body":"lucene is very cool"}
{"index":{}}
{"body":"Elasticsearch builds on top of lucene"}
{"index":{}}
{"body":"Elasticsearch rocks"}
{"index":{}}
{"body":"elastic is the company behind ELK stack"}
{"index":{}}
{"body":"Elk stack rocks"}
{"index":{}}
{"body":"elasticsearch is rock solid"}

//suggest_mode:popular, missing, always
POST /articles/_search
{
	"size": 1,
	"query":{
		"match": {
			"body": "lucen rock"
		}
	},
	"suggest": {
		"term-suggestion": {
			"text":"lucen rock",
			"term": {
				"suggest_mode":"popular",
				"field":"body"
			}
		}
	}
}

//能够对拼写错误的单词返回建议
POST /articles/_search
{
	"suggest": {
		"term-suggestion": {
			"text":"lucen hock",
			"term": {
				"suggest_mode":"always",
				"field":"body",
				"sort":"frequency",
				"prefix_length":0
			}
		}
	}
}

//phrase suggestor
//confidence：限制返回的结果数
POST /articles/_search
{
	"suggest": {
		"my-suggestion": {
			"text":"lucene and elasticsear rock hello world",
			"phrase": {
				"field":"body",
				"max_errors":2,
				"confidence":0,
				"direct_generator": [{
					"field":"body",
					"suggest_mode":"always"
				}],
				"highlight":{
					"pre_tag":"<em>",
					"post_tag":"</em>"
				}
			}
		}
	}
}
```

#### Completion

根据输入的内容自动搜索，补全内容

```json
DELETE articles
PUT articles
{
	"mappings": {
		"properties": {
			"title_completion": {
				"type": "completion"
			}
		}
	}
}

POST articles/_bulk
{"index":{}}
{"title_completion":"lucene is very cool"}
{"index":{}}
{"title_completion":"Elasticsearch builds on top of lucene"}
{"index":{}}
{"title_completion":"Elasticsearch rocks"}
{"index":{}}
{"title_completion":"elastic is the company behind ELK stack"}
{"index":{}}
{"title_completion":"Elk stack rocks"}
{"index":{}}
{"title_completion":"elasticsearch is rock solid"}

POST /articles/_search?pretty
{
	"size":0,
	"suggest": {
		"articles-suggest": {
			"prefix":"el",
			"completion": {
				"field":"title_completion"
			}
		}
	}
}
```

#### Context

根据上下文信息，补全相应的内容

```json
PUT comments
PUT comments/_mapping
{
	"properties": {
		"comment_autocomplete": {
			"type":"completion",
			"contexts": [{
				"type":"category",
				"name":"comment_category"
			}]
		}
	}
}
POST comments/_doc
{
	"comment":"I love the start war movies",
	"comment_autocomplete": {
		"input":["star wars"],
		"contexts": {
			"comment_category":"movies"
		}
	}
}
POST comments/_doc
{
	"comment":"Where can I find a Starbucks",
	"comment_autocomplete": {
		"input":["starbucks"],
		"contexts": {
			"comment_category":"coffee"
		}
	}
}

POST comments/_search
{
	"suggest": {
		"my-suggest": {
			"prefix":"sta",
			"completion": {
				"field": "comment_autocomplete",
				"contexts":{
					"comment_category":"coffee"
				}
			}
		}
	}
}
```

#### 几个Suggester的比较

- 精准度：Completion > Phrase > Term
- 召回率：Term > Phrase > Completion
- 性能：Completion > Phrase > Term



