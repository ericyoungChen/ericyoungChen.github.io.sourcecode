title: 配置Nagios监控服务
date: 2015-12-23 19:50:27
tags: [Nagios,监控]
---

### 一、前言 ###

Nagios的安装以及基本的设置在前一篇文章中已经描述完成。然而打开网页发现仅仅显示的是一个显示Nagios版本号以及一些简介的页面，没有任何有用的信息反馈给我们。本文的目的便是围绕监控服务的搭建来展开。

### 二、Nagios软件结构的介绍 ###

安装完Nagios核心以及插件之后，可以到Nagios目录下去查看它的目录结构。大致如下：

![Nagios目录结构](http://7xpdnd.com1.z0.glb.clouddn.com/nagios.jpg)

<!--more-->
可以看到，Nagios的服务主要在`../nagios/etc`下配置。该目录下有定义主机以及服务的模板文件`templates.cfg`以及其他有关的配置文件。可以根据自己的需要在该目录下建立子目录。所以在配置上有很大的灵活性。

### 三、配置实例 ###

为了方便维护，建议为Nagios各个定义对象创立独立的配置文件：创建`hosts.cfg`文件定义主机和主机组，创建`services.cfg`文件定义服务，用默认的`contacts.cfg` `commands.cfg`定义对应内容。下面分别介绍如下：

* templates.cfg文件
	
	Nagios主要用于监控主机资源以及服务(在此统称为对象)，为了不重复定义一些监控对象，Nagios引入了一个模板配置文件，把一些共性的属性定义成模板，一遍多次引用，这便是`templates.cfg`的作用。它的文件内容部分如下：
			

![t1](http://7xpdnd.com1.z0.glb.clouddn.com/templates1.png)
![t2](http://7xpdnd.com1.z0.glb.clouddn.com/templates1.png)
![t3](http://7xpdnd.com1.z0.glb.clouddn.com/templates3.png)


* commands.cfg

	此文件默认存在，无需修改即可使用，也可以自己添加新的命令。直接在此文件中添加按照它的格式添加即可,下面是部分内容：

![c1](http://7xpdnd.com1.z0.glb.clouddn.com/commands.png)
![t2](http://7xpdnd.com1.z0.glb.clouddn.com/commands.png)

* hosts.cfg

	此文件默认情况下不存在，需要自己创建，该文件主要用来指定被监控的主机地址以及相关的属性信息，一个配置好的示例如下：


		define host{
        use                     linux-server
        host_name               lnmp-web
        alias                   lnmp
        address                 192.168.1.120
        use                     linux-server
        host_name               lnmp
        alias                   lnmp
        address                 192.168.1.120
		}

	该文件创建了192.168.1.120一个远程主机，如果需要创建主机组，则需要添加一个定义为hostgroup的define。

* services.cfg

	该文件默认情况下也不存在，需要手动创建，该文件主要用于定义监控的服务以及主机资源，例如监控http服务、ftp服务、主机磁盘空间系统负载等。一个配置好的示例如下：
	
	
		define service{
        use                     local-service
        host_name               lnmp-web
        service_description     PING
        check_command           check_ping!100.0.20%!500.0.60%
		}

		define service{
        use                     local-service
        host_name               lnmp-we
        service_description     SSH
        check_command           check_ssh
		}

	示例中可以看到check_command是引用commands.cfg的定义

* cgi.cfg(此文件在../nagios/etc下)

	此文件用来控制相关的CGI脚本，如果想在Nagios的Web监控界面执行CGI脚本，例如重启Nagios进程，关闭通知，停止Nagios主机监测等，就需要配置cgi.cfg文件。如果配置Nagios监控界面的用户不为nagiosadmin，就需要在此文件中添加用户执行的权限，需要配置的信息段如下：
		

		authorized_for_system_information=nagiosadmin
		authorized_for_configuration_information=nagiosadmin
		authorized_for_system_commands=nagiosadmin
		authorized_for_all_services=nagiosadmin
		authorized_for_all_hosts=nagiosadmin
		authorized_for_all_service_commands=nagiosadmin
		authorized_for_all_host_commands=nagiosadmin

* nagios.cfg(此文件在../nagios/etc下)

	此文件为Nagios的核心配置文件，所有的对象配置文件必须在这个文件中定义才能够发挥作用，在这里引用对象配置文件即可。


		log_file=/usr/local/nagios/var/nagios.log # 指定日志目录
		cfg_file=/usr/local/nagios/etc/objects/commands.cfg
		cfg_file=/usr/local/nagios/etc/objects/contacts.cfg
		cfg_file=/usr/local/nagios/etc/objects/timeperiods.cfg
		cfg_file=/usr/local/nagios/etc/objects/templates.cfg
		cfg_file=/usr/local/nagios/etc/objects/localhost.cfg
		# 上面部分为默认的，下面部分为自己定义的
		cfg_file=/usr/local/nagios/etc/objects/hosts.cfg
		cfg_file=/usr/local/nagios/etc/objects/services.cfg

### 四、验证配置 ###

在前一部分，已经完成了一个基本的监控配置，当然还有联系人以及监控时段没有提到，可以自行研究下。Nagios自带有监测配置是否正确的命令，只需要通过下面的命令即可完成


		/usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg

得到的监测结果如下：


		Nagios Core 4.1.1
		Copyright (c) 2009-present Nagios Core Development Team and Community Contributors
		Copyright (c) 1999-2009 Ethan Galstad
		Last Modified: 08-19-2015
		License: GPL

		Website: https://www.nagios.org
		Reading configuration data...
   			Read main config file okay...
		Error: Could not find any host matching 'lnmp-web' (config file '/usr/local/nagios/etc/objects/services.cfg', starting on line 10)
		Error: Failed to expand host list 'lnmp-web' for service 'SSH' (/usr/local/nagios/etc/objects/services.cfg:10)
		Error processing object config files!


		***> One or more problems was encountered while processing the config files...

     		Check your configuration file(s) to ensure that they contain valid
     		directives and data defintions.  If you are upgrading from a previous
     		version of Nagios, you should be aware that some variables/definitions
     		may have been removed or modified in this version.  Make sure to read
     		the HTML documentation regarding the config files, as well as the
     		'Whats New' section to find out what has changed.


出现这个情况，提示的是没有可以匹配'lnmp-web'的主机。找到hosts.cfg，将其改为如下即可：


		define host{
        use                     linux-server
        host_name               lnmp-web
        alias                   lnmp
        address                 192.168.1.120
		}


再次运行上面的监测命令，当输出如下表明配置没有问题了：


		$ /usr/local/nagios/bin/nagios -v /usr/local/nagios/etc/nagios.cfg

		Nagios Core 4.1.1
		Copyright (c) 2009-present Nagios Core Development Team and Community Contributors
		Copyright (c) 1999-2009 Ethan Galstad
		Last Modified: 08-19-2015
		License: GPL

		Website: https://www.nagios.org
		Reading configuration data...
   			Read main config file okay...
   			Read object config files okay...

		Running pre-flight check on configuration data...

		Checking objects...
			Checked 10 services.
			Checked 2 hosts.
			Checked 1 host groups.
			Checked 0 service groups.
			Checked 1 contacts.
			Checked 1 contact groups.
			Checked 24 commands.
			Checked 5 time periods.
			Checked 0 host escalations.
			Checked 0 service escalations.
			Checking for circular paths...
		Checked 2 hosts
			Checked 0 service dependencies
			Checked 0 host dependencies
			Checked 5 timeperiods
			Checking global event handlers...
			Checking obsessive compulsive processor commands...
			Checking misc settings...

		Total Warnings: 0
		Total Errors:   0

		Things look okay - No serious problems were detected during the pre-flight check

### 五、Nagios的启动与停止 ###

1. 启动Nagios

* 通过初始化脚本启动Nagios


		$ /etc/init.d/nagios start
			# 或者
		$ service nagios start

* 手工方式启动Nagios


		$ /usr/local/nagios/bin/nagios -d /usr/local/nagios/etc/nagios.cfg


2. 关闭Nagios

* 通过初始化脚本关闭Nagios服务
	

		$ /etc/init.d/nagios stop
			# 或者
		$ service nagios stop


* 通过kill方式关闭Nagios


		$ kill <nagios_pid>

3. 重启Nagios

* 通过初始化脚本启动Nagios


		$ /etc/init.d/nagios reload
			# 或者
		$ service nagios restart


至此,理论上说Nagios监控系统已经成功搭建而且运行起来了，打开网页`你的主机地址/nagios`输入用户名，密码。弹出页面如下：

![error](http://7xpdnd.com1.z0.glb.clouddn.com/nagios.png)	

点击旁边的按钮没有任何反应。

解决办法：


查看Apache错误日志，位置是`/var/log/httpd`下，显示的错误内容如下：


		[Tue Dec 22 00:45:15 2015] [notice] SELinux policy enabled; httpd running as context unconfined_u:system_r:httpd_t:s0
		[Tue Dec 22 00:45:15 2015] [notice] suEXEC mechanism enabled (wrapper: /usr/sbin/suexec)
		[Tue Dec 22 00:45:15 2015] [notice] Digest: generating secret for digest authentication ...
		[Tue Dec 22 00:45:15 2015] [notice] Digest: done
		[Tue Dec 22 00:45:15 2015] [notice] Apache/2.2.15 (Unix) DAV/2 PHP/5.3.3 configured -- resuming normal operations
		[Tue Dec 22 01:05:16 2015] [notice] caught SIGTERM, shutting down
		[Tue Dec 22 01:05:17 2015] [notice] SELinux policy enabled; httpd running as context unconfined_u:system_r:httpd_t:s0
		[Tue Dec 22 01:05:17 2015] [notice] suEXEC mechanism enabled (wrapper: /usr/sbin/suexec)
		[Tue Dec 22 01:05:17 2015] [notice] Digest: generating secret for digest authentication ...
		[Tue Dec 22 01:05:17 2015] [notice] Digest: done
		[Tue Dec 22 01:05:17 2015] [notice] Apache/2.2.15 (Unix) DAV/2 PHP/5.3.3 configured -- resuming normal operations
		[Tue Dec 22 01:22:22 2015] [error] [client 192.168.1.102] user nagiosadmin: authentication failure for "/nagios": Password Mismatch
		[Tue Dec 22 01:22:34 2015] [error] [client 192.168.1.102] (13)Permission denied: exec of '/usr/local/nagios/sbin/statusjson.cgi' failed, referer: http://192.168.1.121/nagios/main.php
		[Tue Dec 22 01:22:34 2015] [error] [client 192.168.1.102] Premature end of script headers: statusjson.cgi, referer: http://192.168.1.121/nagios/main.php
		[Tue Dec 22 01:22:50 2015] [error] [client 192.168.1.102] (13)Permission denied: exec of '/usr/local/nagios/sbin/status.cgi' failed, referer: http://192.168.1.121/nagios/side.php
		[Tue Dec 22 01:22:50 2015] [error] [client 192.168.1.102] Premature end of script headers: status.cgi, referer: http://192.168.1.121/nagios/side.php
		[Tue Dec 22 01:22:55 2015] [error] [client 192.168.1.102] (13)Permission denied: exec of '/usr/local/nagios/sbin/statusjson.cgi' failed, referer: http://192.168.1.121/nagios/main.php
		[Tue Dec 22 01:22:55 2015] [error] [client 192.168.1.102] Premature end of script headers: statusjson.cgi, referer: http://192.168.1.121/nagios/main.php
		[Tue Dec 22 01:43:08 2015] [notice] caught SIGTERM, shutting down
		[Wed Dec 23 19:37:09 2015] [notice] SELinux policy enabled; httpd running as context unconfined_u:system_r:httpd_t:s0

提示的大致内容是SELINUX没有关。所以讲它关闭即可：

		
		$ sed -i 's/SELINUX=*/SELINUX=disable/' /etc/sysconfig/selinux
		$ setenforce 0

重启Nagios即可：

![success](http://7xpdnd.com1.z0.glb.clouddn.com/successnagios.png)
	