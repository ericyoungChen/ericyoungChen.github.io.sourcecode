title : ELasticSearch 初探
date: 2017-07-30 18:00
tags: [ElasticSearch, ES]
---



# ELasticSearch 初探 #

## 一、 背景 ##

1. 扯皮的东西(来自官网)

	* 分布式的实时文档存储，每个字段均可被索引与搜索
	* 分布式实时分析搜索引擎
	* 能胜任上百个服务节点的扩展，并支持 PB 级别的结构化或者非结构化数据

2. 面向文档，所有对象的内容均可索引
3. 交互格式为`JSON`, 非常友好的`RESTFUL`接口
4. 官网提供主流语言API，即使非主流的语言，社区也有维护和开发各自的`API`
5. 一些架构层的概念

	* 节点(Node): 每个`ES`实例为一个节点，节点有三种角色
	* 集群(Cluster): 配置有同一集群名称的节点集合。集群为数据提供高可用的保证
	* 分片(shard): 分片为存储单机容量的数据提供了可能，数据可以被`ES`分发到多个存储节点(即有分片索引的data node)，默认为5个分片，可以在初始化数据的时候进行设置，无法动态更改
	* 副本(replica): 为保证数据的完整性，可以为主分片数据设置副本，主分片出问题的时候可以将副本分片提升为主分片提供服务。

6. 一些数据层概念：

	* 索引(indexing): 在这里，索引为一个**动词**，代表一种将数据存储到`ES`中的这个动作，即相当于`MySQL`中的 `INSERT`
	* 属性(fields): 多个属性，构成一个文档对象，相当于`MySQL`里的 `COLUMN`
	* 文档(document): 即存储在`ES`(`ElasticSearch`简写)里的对象，相当于`MySQL`中的 `ROW`
	* 类型(type): 多个 *文档* 对象构成一个类型， 相当于`MySQL`里的`TABLE`
	* 索引(indices[n]): 这个在中文官网里面也翻译成索引。。。我更倾向于翻译成集合，是有多个类型构成的一个对象，相当于`MySQL`里的`DATABASE`

| Relational DB | Databases | Tables | Rows      | Columns |
| :-----------: | :-------: | :----: | :-------: | :-----: |
| Elasticsearch | Indices   | Types  | Documents | Fields  |


<!--more-->


## 二、 安装 ##

#### 1. 环境要求

	* 非root用户
	* jdk1.8+

#### 2. 下载包，解压
#### 3. 修改配置文件：


		$> grep -v ^# elasticsearch/config/elasticsearch.yml 
		
			 cluster.name: juanpi-goods-cluster 	#指定集群名称
			 node.name: node 						#节点名称
			 path.data: /data/elasticsearch/data	# 数据存放路径
			 path.logs: /data/elasticsearch/logs	#日志路径
			 network.host: 0.0.0.0					#服务ip
			 http.port: 9200						#服务端口，接受http请求
			 transport.tcp.port: 9202				# 集群通讯接口
			 discovery.zen.ping.unicast.hosts: ["192.168.151.9"]	#设置节点初始列表，根据列表里的节点寻找集群信息
			 bootstrap.system_call_filter: false	#沙盒测试参数，centos6不支持，es5.2以后默认开启，在centos6中要将其关闭
			 node.master: true						#设置为Master节点
			 node.data: true						#设置为数据节点
			 node.ingest: true						#索引之前预处理文档对象
			 http.cors.enabled: true				#允许跨域请求
			 http.cors.allow-origin: "*"			#http请求规则   
			 discovery.zen.minimum_master_nodes: (主节点数/2)+1 #设置这个的规则能避免脑裂现象,如果不做合理设置，意外退出集群的节点很可能会形成一个新的集群

#### 4. 配置区分节点角色


* Master Node(主节点，集群控制节点, 决定节点分配，):


			node.master:true
			node.data:false
	
* Data Node(数据节点，保存数据和执行数据相关操作):


			node.master:fales
			node.data: true

* Client Node(客户端节点, 响应用户请求，转发至其他可操作的节点):


			node.master: false
			node.data: false


## 三、基本操作 ##


### 3.1 索引管理: ###


1. 新增


		PUT http://192.168.151.9:9200/test23/  #默认5个分片，1个副本
		{
			"settings" : {
				"index" : {
					"number_of_shards": 6, ##分片数，不大于data节点数*2
					"number_of_replicas":2 ##副本数，不大于data节点数,可动态调整
					}
			}
		}


2. 修改索引分配(副本数目可动态设置，分片无法动态设置)：

		
		PUT http://192.168.151.9:9200/test23/_settings
		{
		"index":{"number_of_shards":6}
		}
				
3. 获取索引信息：

		
		GET http://192.168.151.9:9200/test23
		GET http://192.168.151.9:9200/test23/_aliases
		GET http://192.168.151.9:9200/test23/_setting
		GET http://192.168.151.9:9200/test23/_mapping



4. 开启与关闭索引(关闭后仅能查看索引的元信息无法读写)


		POST http://192.168.151.9:9200/test23/_close #关闭
		POST http://192.168.151.9:9200/test23/_open  #开启


### 3.2 类型管理 ###


1. 新增类型`type`(例子为在`test23`索引里添加名为`human`的`type`,带一个名为`name_field`的`field`类型为`string`):


		PUT http://192.168.151.9:9200/test23/_mapping/human
		{
		    "properties":{
			"name_field":{"type":"string"}
			} 
		}


### 3.3 文档管理 ###

1. 创建一个新的index，新的type，同时插入一个`document`对象：`PUT /index/type/id`


			PUT http://ip:port/test/book/1
			{
			  "first_name": "Douglas",
			  "last_name": "Fir",
			  "age": 35,
			  "about": "I like to build cabinets",
			  "interests": ["forestry"]
			}



2. 直接新增文档

	
		PUT http://192.168.151.9:9200/test23/book/1 (如果直接新增数据，就默认5个分片，1个副本，如果文档本身存在，为一次更新操作，_version字段就会被更新一次)
		{
               "account_number": 450,
               "balance": 2643,
               "firstname": "Bradford",
               "lastname": "Nielsen",
               "age": 25,
               "gender": "M",
               "address": "487 Keen Court",
               "employer": "Exovent",
               "email": "bradfordnielsen@exovent.com",
               "city": "Hamilton",
               "state": "DE"
            }



3. 删(删除数据)


		DELETE http://192.168.151.9:9200/test23/book/1


4. 查

		
		GET http://192.168.151.9:9200/test23/book/1


5. 高级查询


	* 条件查询:

	
			POST /test23/book/_search
			{
			    "query" : {
			        "match": {
			           "age" : 25
			        }
			    }
			}


	* 批量获取数据`_mget`:

			POST 


	
## 四、集群状态监控以及管理 ##



集群的API是`/_cluster/param`，返回的是`JSON`序列，格式化输出的接口是`/_cat/`


* 查看集群状态: 


		GET http://192.168.151.9:9200/_cluster/health
	


* 查看节点信息:


		GET http://192.168.151.9:9200/_nodes


* 查看节点状态:


		GET http://192.168.151.9:9200/_nodes/stats


* 格式化输出接口以及作用如下

URL(?v)|作用
---- | ----
/_cat/health|集群健康状态
/_cat/master|当前的master节点
/_cat/nodes|所有的nodes信息
/_cat/count|所有doc数量
/_cat/aliases|索引别名
/_cat/indices|索引状态以及统计数据
/_cat/shardsshard|分片统计信息
/_cat/allocationshards|分配信息

## 五、 使用场景与意义 ##


1. 由 `ElasticSearch`+`Logstash`+`Kibana` 组成的 `EKL`日志分析组合，用于日志统计分析，数据可视化等
2. 站内搜索，与`solr`竞争
3. 全文搜索： `Wikipedia`用`ES`提供全文搜索，`StackOverflow`使用`ES`结合全文搜索与地理位置查询， `Github`使用`ES`检索代码


