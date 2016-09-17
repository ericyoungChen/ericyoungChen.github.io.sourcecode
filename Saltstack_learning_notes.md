title: Saltstack学习记录
date: 2016-03-08 12:57:16
tags: [saltstack,自动化运维]
---


##### 1、 Grains作用

 用于收集客户机的基本信息，包括：
	
1. 系统版本以及类型
2. 主机名
3. IP地址
4. 内核
5. 系统
6. 内存情况
7. etc...

##### 2、 Grains在salt命令中的使用


在命令`salt -options '<target>' <function> [arguments] `中添加参数`-G`表示运用Grains收集的信息进行匹配，如：

1. `salt -G 'os:Centos' test.ping` 测试操作系统为Centos的机器通信是否正常
2. `salt -G 'cpuarch:x86_64' grains.item num_cup`匹配minion中的64位机器，且返回CPU核心数
3. `salt '*' grains.ls`列出Grains模块列表
4. `salt '*' grains.items`列出Grains模块收集的数据

<!--more-->
##### 3、 在minion的配置文件上调用Grains数据，添加grains选项并且传递参数即可：


		grains:
			#以上部分为minions的top顶层定义，如果将此配置文件放置在minion的/etc/salt/grains下，可以直接使用以下部分
		  roles:
		    - webserver
		    - memcache
		  deployment: datacenter4
		  cabinet: 13
		  cab_u: 14-15

##### 4、 将minion中的文件发送到master：
	
1. 在master配置文件中，设置 file_recv: True(也可以设置传输文件的大小)
2. 修改后重启master
3. 执行命令： salt 'target' --log--level=all cp.push '文件名'
4. 文件复制到默认位置为/var/cache/salt/master/minions下的对应id目录

##### 5、 .sls文件的编写规则
	
1. 遵循YAML语法结构，代码逻辑以缩进区分
2. 缩进：使用缩进来逻辑区分代码块
3. `:`冒号：-每个冒号后面一个空格，如：`my_key: my_key`
4. `-`破折号：每个破折号后加空格，来添加选项列表如：


		- list_value_one
		- list_value_two
		- list_value_three

		或者

		my_dictionary:
		  - list_value_one
		  - list_value_two
		  - list_value_three



##### 6、saltstack配置管理

1. master配置文件下配置`file_roots部分`默认路径为`/srv/salt`
2. 根据需要在`srv/salt`路径下创建文件以及文件夹
3. 创建一个作为入口的`sls`文件：`top.sls`内容如下：


		$> vim /srv/salt/tops.sls

		base: # 指明salt这是基础配置文件
		  'minion1': # 是指应用在id为minion1的客户机上
    		- apache.apache # 是指相关配置在apache目录下的apache.sls
		  

4. 编辑`/srv/salt/apache`下的文件：


		$> vim /srv/salt/apache/apache.sls
		
		apache: 			# 自定义名称，说明这是管理apache的
		  service: 			# 这是服务管理
    		- name: httpd 	# 管理httpd服务
    		- running 		# 要保证上一行的服务是运行状态
    		- restart: True # 告诉salt这个服务一旦挂了需要自动重启
    		- watch: 		# 监视下面一行的文件，若有改动，则执行上一行定义的行为
      		  - file: /usr/local/apache2/conf/httpd.conf # 指定需要监视的文件

		/usr/local/apache2/conf/httpd.conf:    # 文件标志
		  file: 
			- managed 						   # 指明文件管理
    		- source: salt://apache/httpd.conf # 上述需要修改的文件要遵循这里指定的文件
    		- backup: minion 				   # 告诉包管理工具，当文件更新时，在更新前备份一次。

5. 第4步的目录下添加一个作为标准的httpd.conf文件
6. 然后在master端执行`salt 'minion1' state.highstate`将会返回一下内容，表明配置管理完成

		

		$> salt 'minion1' state.highstate
		minion1:
		----------
		          ID: /usr/local/apache2/conf/httpd.conf
		    Function: file.managed
		      Result: True
			 Comment: File /usr/local/apache2/conf/httpd.conf is in the correct state
     		 Started: 00:51:08.307689
    		Duration: 8.097 ms
     		 Changes:   
		----------
				  ID: apache
			Function: service.running
				Name: httpd
			  Result: True
     		 Comment: The service httpd is already running
     		 Started: 00:51:08.315916
    		Duration: 25.544 ms
    		 Changes:   

		Summary
		------------
		Succeeded: 2
		Failed:    0
		------------
		Total states run:     2

7. 只要minion1的配置文件发生改动，master便可监测出来然后以master上的配置文件为模板修改回来或者在master上修改，批量修改minion的配置文件

##### 7、包管理

1. master配置文件下配置`file_roots部分`默认路径为`/srv/salt`
2. 根据需要在`srv/salt`路径下创建文件以及文件夹
3. 在入口`sls`文件：`top.sls`添加如下内容内容：


		$> vim /srv/salt/tops.sls

		base: 
		  'minion1': 
    		- apache.apache 
		  'minion2': #指明这是在id为minion2的客户机上进行的操作
    		- packinstall.pack # 是指相关配置在packinstall目录下的pack.sls

4. 编辑`/srv/salt/packinstall/pack.sls`


		apache: # 表明该文件是管理apache
		  pkg: # 是包管理
			- name: httpd # 安装的软件名称
    		- installed # 需要被安装
		service: # 一下内容与配置管理类似
    		- name: httpd
    		- running
    		- restart: True
    		- watch:
			  - file: /etc/httpd/conf/httpd.conf

		/etc/httpd/conf/httpd.conf:
		  file.managed:
    		- source: salt://packinstall/httpd.conf
    		- user: root
    		- group: root
    		- mode: 644
    		- backup: minion


5. 执行`salt 'minion2' state.highstate`出现如下提示表示配置完成：


				......
		                 +#<VirtualHost *:80>
                  +#    ServerAdmin webmaster@dummy-host.example.com
                  +#    DocumentRoot /www/docs/dummy-host.example.com
                  +#    ServerName dummy-host.example.com
                  +#    ErrorLog logs/dummy-host.example.com-error_log
                  +#    CustomLog logs/dummy-host.example.com-access_log common
                  +#</VirtualHost>
			----------
              ID: apache
    	Function: service.running
        	Name: httpd
      	  Result: True
     	 Comment: Started Service httpd
     	 Started: 00:26:40.589898
    	Duration: 99.403 ms
     	 Changes:   
              ----------
              httpd:
                  True

		Summary
		------------
		Succeeded: 3 (changed=2)
		Failed:    0
		------------
		Total states run:     3