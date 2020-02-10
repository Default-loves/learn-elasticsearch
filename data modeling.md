### 对象和Nested对象

数据存储的时候，内部对象的边界并没有考虑在内，JSON格式被处理成扁平式的键值对结构

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

