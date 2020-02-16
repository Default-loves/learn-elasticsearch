

### Index Pattern

```json
PUT /logstash-2015.05.18
{
  "mappings": {
    "properties": {
      "geo": {
        "properties": {
          "coordinates": {
            "type": "geo_point"
          }
        }
      }
    }
  }
}
PUT /logstash-2015.05.19
{
  "mappings": {
    "properties": {
      "geo": {
        "properties": {
          "coordinates": {
            "type": "geo_point"
          }
        }
      }
    }
  }
}
PUT /logstash-2015.05.20
{
  "mappings": {
    "properties": {
      "geo": {
        "properties": {
          "coordinates": {
            "type": "geo_point"
          }
        }
      }
    }
  }
}
//导入数据
curl -H 'Content-Type: application/x-ndjson' -XPOST 'localhost:9200/_bulk?pretty' --data-binary @logs.jsonl
//导入数据
curl -H 'Content-Type: application/x-ndjson' -XPOST 'localhost:9200/bank/account/_bulk?pretty' --data-binary @accounts.json

//之后可以在Kibana的Management上进行Create Index Pattern

```



### Discover数据

### 构建Dashboard

