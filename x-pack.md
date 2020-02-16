### 用Monitoring和Alerting监控Elasticsearch集群

X-Pack提供了免费集群监控功能

Watcher for Alerting需要Gold账户，不是免费的

在生产环境中，建议搭建dedicated集群来监控Elasticsearch集群，这样能够减少Elasticsearch集群的负载和数据，而其被监控集群出现问题的时候，还能够看到监控相关的数据

在Kibana界面上开启

### 用APM进行程序性能监控

去Elasticsearch官网下载APM Server

如果是连接本地的Elasticsearch则可以直接执行`apm-server`，否则需要配置apm-server.yml，之后可以在Kibana的APM进行Server情况的查看check APM server status

配置AMP Agent（监控的对象），下面以java程序为例

- 下载APM agent

- 使用配置语句启动java程序

- ```
  java -javaagent:/path/to/elastic-apm-agent-<version>.jar \
       -Delastic.apm.service_name=my-application \
       -Delastic.apm.server_url=http://localhost:8200 \
       -Delastic.apm.secret_token= \
       -Delastic.apm.application_packages=org.example \
       -jar my-application.jar
  ```

- 当java程序进行活动的时候，可以在Kibana APM进行监控

使用性能测试工具gatling

- 下载gatling：https://gatling.io
- 下载Scala
- 将测试脚本app.scala放置到gatling/user-files/simulations/junyi/目录下

- 运行gatling.sh

### 用机器学习实现时序数据的异常检测

Elasticsearch的ML，主要针对时序数据的异常检测和预测

ML的模型需要考虑数据的周期时间问题，时间太长的话影响因素比较多，导致随机分布，时间太短的话模型是随机波动的

使用server-metrics数据进行simple metric

使用server-metrics数据进行Multi metric

使用user-activity数据进行Population

Calendar management让Job对某些日期的数据不做处理（比如节假日的数据和平时是有明显偏差的）

### 用ELK进行日志管理

集中化的日志管理过程：日志搜集 -> 格式化分析 -> 搜索和可视化 -> 风险预警

使用Filebeat读取日志文件，其不会对数据做处理，如果需要对数据处理可以使用Logstash或者Ingest Pipeline的方式进行

```json
下载Filebeat

编辑filebeat.yml

开启相应的module：`filebeat modules enable system`

开启Kibana dashboard：`filebeat setup --dashboard`

启动filebeat：`filebeat -e`
```

### 用Canvas做数据演示

更加酷炫的方式，展示实时的数据分析结果