# Learn-Elasticsearch





### 通过Docker安装

```shell
# 拉取镜像安装
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.10.2

# 运行
docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.10.2
```

