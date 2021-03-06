title: 对Twemproxy中一致性hash与hash tags的理解
date: 2016-09-27 20:35:00
tags: [Twemproxy,Redis,hash tags]
---

## 对Twemproxy中一致性hash与hash tags的理解

##### 1. 一致性hash ####

twemproxy中的服务集群既能够以`host:port:weight`形式进行匹配，也可以通过`host:port:weight name`进行匹配,配置文件中的字段如下：


		servers:
		 - 127.0.0.1:6379:1
		 - 127.0.0.1:6380:1
		 - 127.0.0.1:6381:1
		 - 127.0.0.1:6382:1

<!--more-->

或者


		servers:
		 - 127.0.0.1:6379:1 server1
		 - 127.0.0.1:6380:1 server2
		 - 127.0.0.1:6381:1 server3
		 - 127.0.0.1:6382:1 server4


在第一个配置文件中，keys可以直接匹配到`host:port:weight`这一组合中，而后一个配置文件，keys会被匹配到一个将会指向定义的节点上也就是形如`host:port pair`的节点。后一个配置文件给了我们一个重定义节点到一个不同服务的方式而不需要干扰到hash环。所以当 `auto_eject_hosts`的值被设定为`false`的时候，后一种配置的方法更为妥当。

##### 2. Hash Tags

Hash Tags的设定后，只需要匹配部分的key便可计算出hash值。当 `hash tag`被设定的时候，我们使用tag里的部分值来作为一致性hash的值。反之，当`hash tag`没有被设定的时候，我们需要使用所有的key来作为匹配的项目。这项参数的设定只要tag里面的key值是一样的，就能够将他们同时指向一个server。

比如，一个server pool的设定如下。hash_tag的符号设定是`{}`。这个符号的意思是`user:{user1}:ids`和`user:{user1}:tweets`这两个key能够指向同一个server，因为在这里，我们通过`{}`里面的`user1`来作为hash匹配的基准。对于形如`user:user1:ids`,就只能将整个key作为计算hash的基准，因此这可能会导致和`user:user1:tweets`所匹配到的server不一样。


		beta:
		  listen: 127.0.0.1:22122
		  hash: fnv1a_64
		  hash_tag: "{}"
		  distribution: ketama
		  auto_eject_hosts: false
		  timeout: 400
		  redis: true
		  servers:
		    - 127.0.0.1:6380:1 server1
			- 127.0.0.1:6381:1 server2
			- 127.0.0.1:6382:1 server3
			- 127.0.0.1:6383:1 server4
