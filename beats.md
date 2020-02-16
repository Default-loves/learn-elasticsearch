### 总述

是一个轻量级的数据采集器，包括很多开箱即用的比如Filebeat、Metricbeat等

https://www.elastic.co/guide/en/beats/metricbeat/current/index.html

### Metricbeat

用来定期搜集操作系统，软件的指标数据，指标数据存储在Elasticsearch，通过Kibana进行实时的数据分析

组成：

- Module：搜集的指标对象，例如不同的操作系统，不同的数据库，不同的应用程序等
- Metricset：具体的指标集合，以减少调用次数为原则进行划分
- www.elastic.co/guide/en/beats/metricbeat/7.5

```json
从网络上安装metricbeat

查看Module，默认开启了System Module：`metricbeat module list`

开启mysql Module：`metricbeat module enable mysql`

往Kibana添加Dashboard：`metricbeat setup --dashboards`

配置mysql的用户名和密码：`vim metricbeat/modules.d/mysql.yml`

开启MetricBeats：`metricbeat`

对数据库做一些操作，可以在kibana的Dashboard看到指标数据的信息
```



### Packetbeat

实时的网络数据分析，监控应用服务器之间的网络流量

```json
从网络上安装packetbeat
往Kibana添加Dashboard：`packetbeat setup --dashboards`
配置：`vim modules.d/packetbeat.yml`
开启Packetbeat：`packetbeat`
```

