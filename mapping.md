### Mapping

Mapping类似关系数据库中的schema定义，作用是定义索引中字段的名称、字段的数据类型、字段，倒排索引的相关配置



Dynamic Mapping：在写入文档的时候，如果没有索引，那么就会根据文档信息，自动推算出字段的类型



查看Mapping：`GET /movies/_mappings`



Mapping的字段类型更改

- 新增加字段
  - Dynamic设置为true，对于新增加的文档字段，Mappings会更新
  - Dynamic设置为false，Mapping不会更新，无法被索引，但是信息会出现在_source中
  - Dynamic设置为strict，文档写入失败
- 对已有的字段：不支持修改，除非reindex，重建索引



创建Mapping好的实践

1. 创建一个临时的index，写入一些样例数据
2. 通过访问Mapping API，获得Dynamic Mapping的定义
3. 修改后，创建索引
4. 删除临时的index



控制倒排索引记录的内容：设置`"index_options":"docs"/"freqs"/"positions"/"offsets"`



#### Mapping配置项

```json
//设置index：false说明该字段不会被搜索到
DELETE users
PUT users
{
	"mappings": {
		"properties": {
			"firstname": {
				"type": "text"
			},
			"lastname": {
				"type": "text"
			},
			"mobile": {
				"type": "text",
				"index": false
			}
		}
	}
}
PUT /users/_doc/1
{
	"firstname":"Xu",
	"lastname":"Junyi",
	"mobile":"123"
}
//搜索不到的
POST /users/_search
{
	"query":{
		"match":{
			"mobile":"123"
		}
	}
}

//需要对null值进行搜索
//只有keyword类型支持设置null_value
DELETE users
PUT users
{
	"mappings": {
		"properties": {
			"firstname": {
				"type": "text"
			},
			"lastname": {
				"type": "text"
			},
			"mobile": {
				"type": "keyword",
				"null_value": "NULL"
			}
		}
	}
}
PUT users/_doc/1
{
	"firstname":"Xu",
	"lastname":"Junyi",
	"mobile":null
}
PUT users/_doc/2
{
	"firstname":"Feng",
	"lastname":"Wenxing"
}
GET users/_search
GET users/_search
{
	"query": {
		"match": {
			"mobile": "NULL"
		}
	}
}


//copy_to设置，字段聚合在一起
DELETE users
PUT users
{
	"mappings": {
		"properties": {
			"firstname": {
				"type": "text",
				"copy_to":"fullname"
			},
			"lastname":{
				"type": "text",
				"copy_to":"fullname"
			}
		}
	}
}
PUT users/_doc/1
{
  "firstname":"Xu",
  "lastname":"Junyi"
}
GET users/_search?q=fullname:(Xu Junyi)

//Elasticsearch不提供数组类型，但是任何字段都可以包含多个相同类型的数值
PUT users/_doc/1
{
	"name":"one",
	"interests":"reading"
}
PUT users/_doc/2
{
	"name":"two",
	"interests":["reading","music"]
}
POST /users/_search
{
	"query": {
		"match_all":{}
	}
}
//"interests"的type为text，而不是数组
GET /users/_mappings
```

