title: php-fpm多实例
date: 2016-03-10 
tags: [php-fpm,multi-daemon]
---

## php-fpm多实例启动 ##

>这个事情其实应该在几个月前就搞定了，个人原因，说到底还是没有钻研到底的那种精神。经过一段时间的挣扎，也对这部分的知识有了更深刻的理解，多实例的关键在于配置文件以及对应的pid文件还有启动脚本的不冲突。至于配置参数调优这一块，至今还没有去研究，现在有个一个很好的环境，所以得把握好机会，多学习吧。
>下面就是正题，以其中一个实例为例子：

1. 编译安装


		$> yum -y install gcc gcc-c++
		# 安装c和c++编译器
		$> yum -y install libxml2-devel openssl-devel libcurl-devel libjpeg-devel libpng-devel libicu-devel openldap-devel
		# 安装依赖
		$> cd /usr/local/src/php-5.6.19
		$> ./configure --prefix=/usr/local/webserver/php --enable-fpm
		# 选择编译参数
		$> make &&　make install 
		# 安装
		$> cp /usr/local/src/php-5.6.19/sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm
		# 配置启动脚本
		$> chmod +x /etc/rc.d/init.d/php-fpm 
		# 添加执行权限

<!--more-->

2. 编辑多实例对应配置文件


		$> cd /usr/local/webserver/php/etc
		$> cp php-fpm.conf.default php-fpm0.conf
		$> diff php-fpm0.conf php-fpm.conf.default
		25c25
		# 表示25行修改了对应的pid文件
		< pid = run/php-fpm0.pid
		---
		> ;pid = run/php-fpm.pid
		32c32
		# 表示第32行修改了对应的错误日志文件名称
		< error_log = log/php-fpm0.log
		---
		> ;error_log = log/php-fpm.log
		149,150c149,150
		# 表示第149，150行修改了用户和对应组
		< user = www
		< group = www
		---
		> user = nobody
		> group = nobody
		164,165c164
		# 164-165行修改了监听方式以及对象(默认为监听9000端口，改为监听指定的socket文件)
		< ; listen = 127.0.0.1:9000
		< listen=/dev/shm/php-fpm0.socket
		---
		> listen = 127.0.0.1:9000
		178c177
		# 去掉监听方式的注释
		< listen.mode = 0660
		---
		> ;listen.mode = 0660
		225c224
		# 将运行php-fpm进程数量设置为固定值
		< pm = static
		---
		> pm = dynamic
		442c441
		# 慢日志指定路径
		< slowlog = /opt/logs/php/slow/$pool.log.slow
		---
		> ;slowlog = log/$pool.log.slow
		448c447
		# 请求慢日志延时
		< request_slowlog_timeout = 5s
		---
		> ;request_slowlog_timeout = 0
		455c454
		


3. 配置启动脚本


		$> cp /etc/init.d/php-fpm /etc/init.d/php-fpm0
		$>　vim /etc/init.d/php-fpm
		# 在参数定义这一块，将原来的配置文件和pid文件修改为对应的配置文件和pid文件即可
		prefix=/usr/local/webserver/php
		exec_prefix=${prefix}
		php_fpm_BIN=${exec_prefix}/sbin/php-fpm		
		php_fpm_CONF=${prefix}/etc/php-fpm0.conf
		# 指定配置文件
		php_fpm_PID=${prefix}/var/run/php-fpm0.pid
		# 指定pid文件
		:wq
		# 保存并退出
		$> chmod +x /etc/init.d/php-fpm
		$> /etc/init.d/php-fpm start
		$> Starting php-fpm  done
		# 至此，以php-fpm0.conf为配置文件，以php-fpm0.pid为pid文件的php实例配置完成

4. 检测搭建情况

		$> pstree -p
		
![php多实例图](http://7xpdnd.com1.z0.glb.clouddn.com/Day6.png) 

		