    title: Consul, Consul-Template 进行配置管理
    date: 2016-11-1 20:35:00
    tags: [Consul,Consul-Template,Config-manager]
    ---
    
## 运用Consul,Consul-Template进行配置管理 ##

Edited by houzi


### 一、背景 ###

对于运行在测试环境的代码，根据不同的测试场景以及测试环境，对代码的环境变量从代码中解耦出来，脱离代码管理，而增加配置的可变更性，对于运维以及开发而言，对资源变更操作所需的成本更低。再配合内网DNS解析，规范域名命名原则，结合`hosts`文件的下发从而对配置进行无差异化管理。


### 二、现状 ###

现有测试环境多套，在一处测试环境对环境变量文件进行修改，完成后进行多处下发，通过`rsync`或者`ansible`或者`saltstack`等同步工具下发。而差异的环境通过脚本进行对配置修改，从而达到配置正确适应该环境的目的。


### 三、解决办法 ###

#### 1. 最初做法:

* 根据开发提供的测试值，运用脚本生成配置文件，直接添加。无重复检查
* 对生成脚本进行重复性检查，重复与非重复变量进行区分。
* 人工对多套环境进行重复copy操作

#### 2. 通过`inotify+rsync`来进行同步管理

坑：

看似方案可行，但有以下几处坑，

* 每次对单个文件进行监控无法触发`inotify`(因为同步过去文件的`inode`已改变)
* 对文件夹内文件进行监控无法收敛。造成触发一秒内多次`reload`php-fpm进程导致服务挂了
* `rsync`的配置繁琐。(应该是我使用的姿势不对)


#### 3. 通过`ansible`或者`saltstack`工具


线上的环境变量，在开始的时候运用的是`saltstack`的`state`模块进行下发以及管理的。相信熟悉`saltstack`工具的同学应该很清楚这个模块的作用。定义好`sls`文件，有环境变量变更先对文件备份，以便快速回滚。

但是测试环境经常涉及到配置查询，有时也涉及到变更。对不同环境的变量也有区分。而没选择这两个工具的更大原因应该是自己想跳出这两个工具的限制(测试环境新增域名已通过`ansible-playbooks`进行管理)。

#### 4. 新的发现

##### a. Consul+Consul-Template 简介

在前辈的偶然提示下，了解到了一个具有`key/value`存储功能的服务发现软件(系统？工具？)`consul`，而且除了键值存储，还有健康检查，融合数据中心等功能。

>`Consul-Template`可以通过监听本地的`Consul`服务来获取注册信息进而完成配置的更新。而且这两个服务部署便捷，仅仅是两个二进制包，解压之后添加到环境变量里，就可以直接调用，对于我们运维而言，仅仅是两个命令而已。还有个优点是自带web界面窗口，带有`ACLs`策略，以及简单的访问权限控制。

![http://7xpdnd.com1.z0.glb.clouddn.com/Consul_a.png](http://7xpdnd.com1.z0.glb.clouddn.com/Consul_a.png)


##### b. `Consul`部署方法


1. 到[https://www.consul.io/downloads.html](https://www.consul.io/downloads.html)选择合适版本下载`Consul`的二进制包
2. 从此处获得`Consul`的Web-ui：[https://releases.hashicorp.com/consul/0.7.0/consul_0.7.0_web_ui.zip](https://releases.hashicorp.com/consul/0.7.0/consul_0.7.0_web_ui.zip)


包安装完，添加至环境变量之后，便可使用命令行了。

##### c. `Consul`命令说明以及集群搭建




###### 命令参数说明


		$> consul -h
		usage: consul [--version] [--help] <command> [<args>]
		
		Available commands are:
		    agent          Runs a Consul agent  #以agent方式启动consul实例，若加-server参数，则表明该节点为server集群之一。
		    configtest     Validate config file
		    event          Fire a new event
		    exec           Executes a command on Consul nodes
		    force-leave    Forces a member of the cluster to enter the "left" state
		    info           Provides debugging information for operators
		    join           Tell Consul agent to join cluster # 告诉改实例要加入的集群
		    keygen         Generates a new encryption key
		    keyring        Manages gossip layer encryption keys
		    leave          Gracefully leaves the Consul cluster and shuts down # 退出集群并且关闭
		    lock           Execute a command holding a lock
		    maint          Controls node or service maintenance mode #将某节点挂成维护状态
		    members        Lists the members of a Consul cluster # 列出集群中的成员
		    monitor        Stream logs from a Consul agent
		    operator       Provides cluster-level tools for Consul operators
		    reload         Triggers the agent to reload configuration files
		    rtt            Estimates network round trip time between nodes
		    version        Prints the Consul version
		    watch          Watch for changes in Consul #监视集群中的变化


以server启动一个节点(一般server节点是3-5个):


		
		$> nohup consul agent -server -bootstrap -dc jp-test -ui-dir=/data/consul.d/ -data-dir=/data/consul.d/data -config-dir=/data/consul.d/conf -pid-file=/data/consul.d/consul.pid -client=0.0.0.0 -advertise=192.168.143.171 -node=192.168.143.171 -rejoin >> /data/consul.d/consul.log &


也可以将上述参数写到一个配置文件中，启动的时候指定该配置文件：

		
		$> cat /data/consul.d/conf/main.json 
		{
		        "bootstrap": true,
				"server": true,
		        "datacenter": "jp-test",
		        "data_dir": "/data/consul.d/data",
		        "pid_file": "/data/consul.d/consul.pid",
				"ui_dir": "/data/consul.d/ui"
		        "client_addr": "0.0.0.0",
		        "node_name": "juanpi-test-171",
		        "rejoin_after_leave": true,
		        "log_level": "INFO",
		        "advertise_addr":"192.168.143.171"
		}

		$> consul agent -config-dir=/data/consul.d/conf


参数解释：

		-server     #表明该节点以server模式运行，
		-bootstrap  # 用来控制一个server是否在bootstrap模式，在一个datacenter中只能有一个server处于bootstrap模式，当一个server处于bootstrap模式时，可以自己选举为raft leader
		-pid-file   #提供一个路径来存放pid文件，可以使用该文件进行SIGINT/SIGHUP(关闭/更新)agent
		-client     #consul绑定在哪个client地址上，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1
		-advertise  #通知展现地址用来改变我们给集群中的其他节点展现的地址，一般情况下-bind地址就是展现地址
		-join       #加入一个已经启动的agent的ip地址，可以多次指定多个agent的地址。如果consul不能加入任何指定的地址中，则agent会启动失败，默认agent启动时不会加入任何节点。
		-node       #节点在集群中的名称，在一个集群中必须是唯一的，默认是该节点的主机名
		-rejoin     #使consul忽略先前的离开，在再次启动后仍旧尝试加入集群中。
		-dc         #该标记控制agent允许的datacenter的名称，默认是dc1
		-ui-dir     #指定web-ui静态文件所在目录，可通过浏览器访问ip:port看到图形界面




###### 构建集群(另外启动多个节点，join到已有节点):


1. 在118机器上执行(以server状态启动):


		$> nohup consul agent -server \
		-dc jp-test \
		-ui-dir=/data/consul.d/ui \
		-data-dir=/data/consul.d/data \
		-config-dir=/data/consul.d/conf \
		-pid-file=/data/consul.d/consul.pid \
		-client=0.0.0.0 \
		-advertise=192.168.143.118 \
		-join=192.168.143.171 \
		-node=192.168.143.118 \
		-rejoin  >> /data/consul.d/consul.log &
	


2. 在117机器上执行(以server状态启动)：


		
		$> nohup consul agent -server \
		-dc jp-test \
		-ui-dir=/data/consul.d/ui \
		-data-dir=/data/consul.d/data \
		-config-dir=/data/consul.d/conf \
		-pid-file=/data/consul.d/consul.pid \
		-client=0.0.0.0 \
		-advertise=192.168.143.117 \
		-join=192.168.143.171 \
		-node=192.168.143.117 \
		-rejoin  >> /data/consul.d/consul.log &



3. 在119机器上执行(未以server状态启动，即为client模式)；

	
		$> nohup consul agent -dc jp-test \
		-data-dir=/data/consul.d/data \
		-config-dir=/data/consul.d/conf \
		-pid-file=/data/consul.d/consul.pid \
		-client=0.0.0.0 \
		-advertise=192.168.143.119 \
		-join=192.168.143.171 \
		-join=192.168.143.117 \
		-join=192.168.143.118 \
		-node=192.168.143.119 -rejoin >> /data/consul.d/consul.log &


4. 至此，一个包含3个`server`节点的`consul`集群搭建完成，通过命令可以查看节点成员状态：


		$> consul members list
		Node             Address               Status  Type    Build  Protocol  DC
		192.168.143.117  192.168.143.117:8301  alive   server  0.7.0  2         jp-test
		192.168.143.118  192.168.143.118:8301  alive   server  0.7.0  2         jp-test
		192.168.143.119  192.168.143.119:8301  alive   client  0.7.0  2         jp-test
		192.168.143.171  192.168.143.171:8301  alive   server  0.7.0  2         jp-test


5. 通过页面访问内容如下:


![consul_nodes](http://7xpdnd.com1.z0.glb.clouddn.com/consul_nodes.png)

该页面可以看到有以下功能

* 节点信息
* 服务节点的健康状态
* `key/value`的存储内容
* `ACL`访问控制表的功能


###### 键值设置

consul的使用，在配置管理这块，能够用到的功能就是它的`key/value`存储功能。而设置键值参数。，有两种方式可以添加。


1. 通过web界面，可以点击`KEY/VALUE`按钮就能看到如下，它支持创建不同的`folder`对键值对进行管理。类似于数据库范畴的表的单位。
2. 通过`http`接口，对其`PUT`参数


对于第一种方式，在此不做细说，看下图应该就能知晓。
![consul_kv](http://7xpdnd.com1.z0.glb.clouddn.com/consul_kv.png)


通过http接口设置键值，则可以通过一些web工具传递`PUT`参数进行设置，如在`shell`下用`curl`工具：


			$> curl -X PUT -d "test" 'http://192.168.143.117:8500/v1/kv/jpenv/test111?dc=jp-test'

`url`参数详解：

* `http://192.168.143.117:8500/v1/kv/`为设置`kv`的地址
* `jpenv`表示`key`要存在哪个定义的文件夹下
* `test111`则是`key`的名称
* `dc=jp-test`表明要存在哪个`datacenter`下
* `test`这个`PUT`传递过去的参数则是`vaule`的值

上面那条指令意思是是在刚才搭建的集群内，设置一个`key`为`test111`在`jp-test`数据中心内`jpenv`目录下而且`value`为`test`

更多的web接口可以访问

[https://my.oschina.net/guol/blog/353394](https://my.oschina.net/guol/blog/353394)

[https://www.consul.io/docs/agent/http.html](https://www.consul.io/docs/agent/http.html)


**到这里可能要问，现在只是提供键值存储，是的，数据有了，但是要怎样生成配置文件？？对的，那就是接下来要介绍的可以根据`consul`中存储的信息能生成文件的`Consul-Template`**

##### d. `Consul-Template`使用


`Consul-Template`顾名思义，是生成需要模版来生成文件用的。有自带的变量，对应`consul`集群中存储的信息。模版文件的编写.

1. 命令使用实例:


		$> cat test.ctmpl 
		{{range ls "jpenv@jp-test"}}
		env[{{.Key}}]='{{.Value}}'{{end}}

模版的编写从`{{range}}`到`{{end}}`的中间部分表明要遍历的数据范围(可依据环境设置不同的路径或者数据中心，从而生成差异性的配置文件)，而写死的则是格式。


根据此模版启动一个`consul-template`实例:


		$> consul-template -consul 127.0.0.1:8500 -wait=3s -template /data/consul.d/info/test.ctmpl:/data/consul.d/info/test.result:date >> template.log & 
		$> cat test.result
		
		env[act_environment]='test'
		env[db_master_biguanxingtai_000_drools_host]='db-master-biguanxingtai-000.jp'
		env[db_master_biguanxingtai_000_drools_port]='3306'
		env[db_master_biguanxingtai_000_drools_pwd]='pnY65kQw'
		env[db_master_biguanxingtai_000_drools_user]='develop'
		env[db_slave_order_pop0_001_host]='192.168.143.233'
		env[db_slave_order_pop0_001_pwd]='juanpi'
		env[db_slave_order_pop0_001_user]='juanpi'
		env[db_slave_order_pop1_001_host]='192.168.143.233'
		env[db_slave_order_pop1_001_port]='3306'
		env[db_slave_order_pop1_001_pwd]='juanpi'
		env[db_slave_order_pop1_001_user]='juanpi'
		env[db_slave_order_user2_001_port]='3306'
		env[db_slave_order_user2_001_pwd]='juanpi'
		env[db_slave_order_user3_001_host]='192.168.143.233'
		env[db_slave_order_user3_001_port]='3306'
		env[db_slave_order_user3_001_pwd]='juanpi'
		env[db_slave_order_user3_001_user]='juanpi'
		env[test111]='test'
		env[testfrompy]='alsofrompy'
		env[twemproxy_marketing_activity_000_port]='9210'



2. 部分参数详解


	* `-consul`指定一个consul服务地址
	* `-template`设定模版文件路径，生成的目标文件路径以及生成文件之后需要执行的操作
	* `-wait`指定生成文件后触发操作的延时时间


在web界面对`key/value`进行操作，就能触发启动命令最后一部分的`shell`命令。(把reload的操作写在这里)