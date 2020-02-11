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