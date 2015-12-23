title: Nagios-运维监控利器
date: 2015-12-21 15:35:10
tags: [Nagios,Apache]
---

### 一、关于Nagios ###

Naigos是一款Linux上成熟的监视系统运行状态和网络信息的开源IT基础设施监视系统。Nagios能监视所指定的本地或者远程主机以及服务，同时提供异常通知，事件处理等功能。

Nagios可以运行在Linux和Unix平台上，同时提供一个可选的基于浏览器的Web界面，便可以方便系统管理员查看系统的运行状态、网络状态、各种系统问题以及日志等。

<!--more-->
### 一、准备工作 ###

* 安装Apache和PHP
		

		$ yum install -y httpd php php-cli gcc glibc glibc-common gd gd-devel net-snmp unzip zip

* 开启Apache服务


		$ service httpd


* 设置一个运行nagios的账户和密码


		$ useradd nagios
		$ passwd nagios


* 设置一个名为nagcmd的用户组，将nagios用户添加到此用户组下


		$ groupadd nagcmd
		$ usermod -a -G nagcmd nagios
		$ usermod -a -G nagcmd apache


### 二、Nagios核心组件的安装 ###

##### 1、源码下载 #####

在sourceforge上下载Nagios-core的源码包

		$ cd /usr/local/src
		$ wget -c http://nchc.dl.sourceforge.net/project/nagios/nagios-4.x/nagios-4.1.1/nagios-4.1.1.tar.gz


##### 2、解压并且安装 #####


		$ tar zxvf nagios-4.1.1.tar.gz && cd nagios-4.1.1
		$ ./configure --with-command-group=nagcmd
		$ make all
		$ make install 
		$ make install-init
		$ make install-config
		$ make install-commandmode
		$ make install-webconf #配置Nagios支持Apache 

##### 3、配置Apache用户以及密码 #####

此步骤的目的是为了设置一个登录Nagios的web界面的用户。


		$ htpass -c /usr/local/nagios/etc/htpasswd.users nagiosadmin

重启Apache服务

		$ service httpd restart


### 三、安装Nagios插件 ###


安装完Nagios核心服务，实现的仅仅是个监控的框架，具体需要监控什么内容，还需要Nagios Plufins的支持

* 下载插件源码并且解压


		$ cd /usr/local/src
		$ wget -c http://www.nagios-plugins.org/download/nagios-plugins-2.1.1.tar.gz
		$ cd nagios-plugins-2.1.1

* 编译安装


		$ ./configure --with-nagios-user=nagios --with-nagios-group=nagios
		$ make && make install

* 验证Nagios配置文件并且启动nagios服务


		$ /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg
		$ 此处出现一堆显示Nagios状态文字
		$ service nagios start

* 将Nagios服务添加到随系统启动


		$ chkconfig --add nagios
		$ chkconfig nagios on 

上述步骤完成之后，可以在Apache的配置目录`/etc/httpd/conf.d/`中找到`nagios.conf`至此，直接打开浏览器，输入***ip地址+/nagios/***即可跳到登录nagiosweb界面

### 四、后记 ###

输入上述的url之后，没有打开期待的页面，查看Apache运行的端口号`netstate -ltnp | grep "httpd"`显示运行在80端口。查看防火墙配置如下

		$ cat /etc/sysconfig/iptables
			# Firewall configuration written by system-config-firewall
			# Manual customization of this file is not recommended.
			*filter
			:INPUT ACCEPT [0:0]
			:FORWARD ACCEPT [0:0]
			:OUTPUT ACCEPT [0:0]
			-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
			-A INPUT -p icmp -j ACCEPT
			-A INPUT -i lo -j ACCEPT
			-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
			-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT # 这一行是后面加上的
			-A INPUT -j REJECT --reject-with icmp-host-prohibited
			-A FORWARD -j REJECT --reject-with icmp-host-prohibited
		COMMIT

发现只开放了22端口，所以在开放22端口的下一行添加开放80端口的语句即可然后重启防火墙`$ service iptables restart`

再次刷新地址，出现如下图片，至此，Nagios+Nagios-Plugins搭建完成。下一篇博文会介绍具体如何配置Nagios监控

![Nagios监控首页](http://7xpdnd.com1.z0.glb.clouddn.com/nagios.png)