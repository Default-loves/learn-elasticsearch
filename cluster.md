### 集群

- 集群的名字设置：`-E cluster.name=junyi`
- 每个节点启动后默认就是一个Master Eligible节点，说明该节点可以参加选主流程
- 第一个节点启动的时候，会成为Master节点
- 每个节点上都保存了集群的信息(节点的信息、索引和其Mapping和Settings等)，只有Master节点可以修改集群的信息
- Master节点：处理创建、删除索引等请求/决定分片被分配到哪个节点/负责索引的创建和删除
- DataNode保存了分片数据
- Coordinating Node负责接收Client的请求，将请求发送到合适的节点，最终将结果进行汇总，每个节点默认都起到了Coordinating Node的职责，通过将其他类型设置为false，使节点变为coordinating node
- 一个节点默认是Master Eligible，Data Node，Ingest Node，当这三个相关的配置参数都是false的时候，节点为Coordinating Node
- 查看集群的健康状况：`GET _cluster/health`

- cerebro: `localhost:9000`



### 分片

- 主分片（Primary Shard）：解决数据水平扩展的问题，数据可以分布在所有的集群中。主分片数在索引创建的时候指定了，后续不能修改，除非reindex
- 副本（Replica Shard）：解决数据高可用的问题
- Elasticsearch的一个分片是Lucene的一个Index。
- Lucene中，一个Segment是一个倒排索引文件，是自包含的，不可修改的，多个Segment汇总在一起称为Index。当有新文档写入的时候，会生成新的Segemnt。Lucene有一个文件叫做Commit Point用来记录所有的Segment的信息，“.del”文件保存了删除文档的信息

#### 分片数

- 分片数过小：会导致后续无法增加节点实现水平扩展；单个分片数据量很大，导致数据重新分配耗时
- 分片数过大：会影响搜索结果的相关性打分，影响统计结果的准确性；浪费资源；影响性能；

查看分片：`GET _cat/shards`

#### Refresh

- 当Index Document的时候，会将Document放到Index Buffer中，同时也会写入Transaction log（为了保证数据不丢失）。将Index Buffer写入到Segment的过程称为Refresh。Refresh后Index Buffer清空，Transaction log不清空
- Refresh不执行fsync操作，Trandaction log会落盘，每个分片有一个Transaction log
- Refresh频率是1s一次，refresh后，文档就可以被搜索到，因此Elasticsearch也被称为近实时搜索
- Index Buffer满了之后也会触发Refresh

#### Flush

1. 调用Refresh
2. 调用fsync，将缓存中的Segment写入磁盘
3. 清空Transaction Log

- 默认30分钟调用1次，或者Transaction Log满的时候(默认512MB)

#### Merge

- 合并Segment，从而减少Segment和删除已经删除的文档
- Elasticsearch和Lucene会自动进行Merge操作，也可以手动执行`POST my_index/_forcemerge`



### 分布式查询和相关性算分

Elasticsearch的查询会分两步：Query和Fetch
- Query：用户将请求发送给Coordinating Node，其会随机在主副分片中选取分片，将请求分发给选中的分片；被选中的分片执行查询，并且排序，每个分片返回from+size个排序后的文档ID和排序值给Coordinating Node
- Fetch：Coordinating Node将获取的文档重新排序，选取From到From+Size个文档的id；以multi get请求的方式，到相应的分片中获取文档的详细的信息

#### 问题

- 每个分片需要查询的文档个数=from+size
- 最终Coordinating Node需要处理的文档个数=number_of_shard * (from+size)
- 深度分页
- 相关性算分在分片之间是独立的，当文档数很少，主分片比较多的时候，相关性算分不准

#### 解决算分不准的方法

- 将主分片设置为1，前提是数据量不大
- 当数据量大的时候，要保证文档均匀分布在所有的分片上
- 使用DFS Query Then Fetch：到每个分片把各分片的词频和文档频率进行搜索，然后进行一次完整的相关性算分，会耗费很多CPU和内存；`_search?search_type=dfs_query_then_fetch`

### 多集群

单集群在水平扩展的时候，节点数不能无限增加，单个Active Master会成为集群的瓶颈，导致整个集群无法工作

```json
//启动3个集群，每个集群只有一个节点
elasticsearch -E node.namme=cluster0node -E cluster.name=cluster0 -E path.data=cluster0_data -E discovery.type=single-node -E http.port=9200 -E transport.port 9300
elasticsearch -E node.namme=cluster1node -E cluster.name=cluster1 -E path.data=cluster1_data -E discovery.type=single-node -E http.port=9201 -E transport.port 9301
elasticsearch -E node.namme=cluster2node -E cluster.name=cluster2 -E path.data=cluster2_data -E discovery.type=single-node -E http.port=9202 -E transport.port 9302


PUT _cluster/settings
{
	"persistent": {
		"cluster": {
			"remote": {
				"cluster0": {
					"seeds": ["127.0.0.1:9300"],
					"transport.ping_schedule":"30s"
				},
				"cluster1": {
					"seeds" ["127.0.0.1:9301"],
					"transport.compress":true,
					"skip_unavailable":true
				},
				"cluster2": {
					"seeds": ["127.0.0.1:9302"]
				}
			}
		}
	}
}

将设置发送到每个集群中
curl -XPUT "http://localhost:920X/_cluster/settings" -H 'Context-Type:application/json' -d'...'

测试数据
curl -XPOST "http://localhost:9200/users/_doc" -H 'Context-Type:application/json' -d'
{"name":"user1", "age":10}'
curl -XPOST "http://localhost:9201/users/_doc" -H 'Context-Type:application/json' -d'
{"name":"user2", "age":20}
curl -XPOST "http://localhost:9202/users/_doc" -H 'Context-Type:application/json' -d'
{"name":"user3", "age":30}

查询
GET /users, cluster1:users, cluster2:users/_search
{
	"query": {
		"range": {
			"age": {
				"gte":10,
				"lte":40
			}
		}
	}
}
```

##### 一个集群多个节点

```json
elasticsearch -E node.name=node1 -E cluster.name=junyi -E path.data=node1_data -E http.port=9200
elasticsearch -E node.name=node2 -E cluster.name=junyi -E path.data=node2_data -E http.port=9201
```

### 脑裂

由于网络连接的问题，一个集群中出现了两个active master节点，维护了不同的cluster state

解决：

可以设置quorum参数，只有当集群中master eligiable节点的数量大于quorum的时候，才进行选举，一般设置quorum的值为（master节点总数 / 2）+1，在Elasticsearch7开始，无需考虑脑裂问题



