安全的基本需求：

- 身份认证
- 权限管理
- 传输加密
- 日志审计

一些方案

- 设置Nginx反向代理
- 安装免费的Security插件
- X-Pack的Basic版本



### 开启并配置X-Pack的认证和鉴权

开启x-pack：`elasticsearch -E node.name=node0 -E cluster.name=junyi -E path.data=node0_data -E http.port=9200 -E xpack.security.enabled=true`

创建默认的用户和分组，并且设置密码：`elasticsearch-setup-passwords interactive`

修改kibana/conf/kibana.yml文件，设置用户名和密码

### 集群内的安全通信

为节点创建证书，证书有3个不同的级别：

- certificate：节点加入需要使用相同CA签发的证书
- full verification：节点加入需要使用相同CA签发的证书，还需要验证host name和ip地址
- no verification：任何节点都可以加入集群，主要用于开发环境

具体操作：

```
生成ca的证书文件elastic-stack-ca.p12：elasticsearch-certutil ca
生辰elastic-certificate.p12并且复制放置到certs文件夹中：elasticsearch-certutil cerrt --ca elastic-stack-ca.p12

使用生成的证书文件启动节点，对于无证书启动的节点会报错
elasticsearch -E node.name=node0 -E cluster.name=junyi -E path.data=node0_data -E http.port=9200 -E xpack.security.enabled=true -E xpack.security.transport.ssl.enable=true -E xpack.security.transport.ssl.verification_mode=certificate -E xpack.security.transport.ssl.keystore.path=certs/elastic-certificates.p12 -E xpack.security.transport.ssl.truststore.path=certs/elastic-certificates.p12
```

### 集群与外部间的安全通信

为Elasticsearch配置HTTPS

```
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/elastic-certificates.p12
xpack.security.http.ssl.truststore.path: certs/elastic-certificates.p12
```



配置Kibana连接Elasticsearch Https

```
- 根据elastic-certificates.p12生成elastic-ca.pem文件，放置到elasticsearch/config/certs目录下：
openssl pkcs12 -in elastic-certificates.p12 -cacerts -nokeys -out elastic-ca.pem
- 配置kibana.yml文件：
elasticsearch.hosts: ["https://localhost:9200"]
elasticsearch.ssl.certificateAuthorities: ["XXXXX/elasticsearch/config/certs"]
elasticsearch.ssl.verificationMode: certificate
```



配置使用HTTPS访问Kibana

```
- 生成一个pem的证书，会生成一个zip文件，将文件解压后放置到kibana/config/certs目录下：elasticsearch-certutil ca --pem
- 配置kibana.yml文件：
server.ssl.enabled: true
server.ssl.certificate: config/certs/ca.crt
server.ssl.key: config/certs/ca.key
```

