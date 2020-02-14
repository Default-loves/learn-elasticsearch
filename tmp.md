需求
将数据库中数据同步到Elasticsearch，借助Elasticsearch的全文搜索，提高搜索速度

- 需要把新增用户信息同步到Elasticsearch中
- 用户Update后，需要被更新到Elasticsearch中
- 用户注销后，Elasticsearch不能搜索到
- 支持增量更新





将mysql-connector-java-8.0.17.jar放置到logstash/logstash-core/lib/jars目录下

logstash -f mysql-demo.conf

