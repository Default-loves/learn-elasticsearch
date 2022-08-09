

`ELK`通常来说指的是三个中间件，分别是`Elasticsearch`、`Logstash`、`Kibana`

下面我们使用docker搭建三个中间件

```shell
# Elasticsearch
docker run -p 9200:9200 -p 9300:9300 --name elasticsearch \
-e "discovery.type=single-node" \
-e "cluster.name=elasticsearch" \
-e "ES_JAVA_OPTS=-Xms512m -Xmx1024m" \
-v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \
-d elasticsearch:7.17.3

chmod 777 /mydata/elasticsearch/data/

# Logstash
docker run --name logstash -p 4560:4560 -p 4561:4561 -p 4562:4562 -p 4563:4563 \
--link elasticsearch:es \
-v /mydata/logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
-d logstash:7.17.3

# 进入容器内部，安装json_lines插件
docker exec -it logstash /bin/bash
logstash-plugin install logstash-codec-json_lines

# Kibana
docker run --name kibana -p 5601:5601 \
--link elasticsearch:es \
-e "elasticsearch.hosts=http://es:9200" \
-d kibana:7.17.3
```

