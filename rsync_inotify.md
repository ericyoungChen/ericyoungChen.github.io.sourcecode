## 利用rync+inotify工具进行文件同步

Edited by houzi

### 一、背景以及意义 ###

> 背景: php代码的环境变量方案实施后，若有新的环境变量添加，则需要对多套功能测试环境以及回归测试环境进行多次修改，而且手动到每台机器修改过程难免会繁琐，而且很难保证每次修改的配置文件一致性。

> 意义: 故将其中一个功能测试环境环境作为rsync服务端，让其他对应测试环境进行修改完环境变量之后进行一次文件同步操作，再对该文件进行监控，若有改动则自动重新加载配置文件，实现一个简单的自动化环境变量批量下发和部署的功能

### 二、简介 ###

#### 1.rsync ####

描述: rsync是一个远程数据同步工具，通过网络快速同步多台主机间的文件。通过增量同步，对于大数据量的文件可以节约网络资源。只针对增加的内容进行备份。

缺点: 虽然rsync的同步能力很强大，但是它仅仅只能做同步的事情，它无法监测是否需要同步以及什么时候同步。这个时候就需要下一个工具的协助`inotify`


#### 2. inotify ####

其实将`inotify`描述为工具，感觉并不是很严谨，确切的说，`inotify`是一个文件系统的事件监控机制。它允许监控程序打开一个独立的文件描述符，并针对文件变更时间对其进行监控，比如文件内容，属性更改等。可以利用这一特性，编写脚本来对要同步的文件进行监测，若该文件有改动则触发需要进行的操作(比如本文写到的重新加载php配置文件的操作)。

Centos6已支持这一特性，执行`ll /proc/sys/fs/inotify`若返回如下结果，则证明支持:

		>$ ll /proc/sys/fs/inotify
		total 0
		-rw-r--r-- 1 root root 0 Sep 18 16:11 max_queued_events
		-rw-r--r-- 1 root root 0 Sep 18 16:11 max_user_instances
		-rw-r--r-- 1 root root 0 Sep 18 16:11 max_user_watches

### 二、功能探索 ###

#### 1. rsync 配置

1. 客户端安装rsync后，在`/etc/`下编辑配置文件，内容如下:

		$> cat /etc/rsyncd.conf
		uid             = root
		gid             = root
		use chroot      = no
		max connections = 200
		timeout         = 600
		pid file        = /var/run/rsyncd.pid
		lock file       = /var/run/rsync.lock
		log file        = /var/log/rsyncd.log
		.
		.
		[php_env]
		path		= /usr/local/webserver/php/etc/fpm.d 
		ignore errors
		read only	= yes
		list		= true
		hosts allow	= many-ips
		hosts deny	= 0.0.0.0/32
		auth users	= rsync
		secrets file    = /etc/rsyncd.passworda
		uid            = rsync
		gid            = rsync


2. 以进程方式启动rsync服务


		$> /usr/bin/rsync --daemon --config=/etc/rsyncd.conf


2. 认证配置:


	* 在服务端上保存认证文件，权限如下:

			$> ll /etc/rsyncd.passworda
			-rw------- 1 root root 39 Sep 18 17:12 /etc/rsyncd.passworda

	* 以及客户端的机器上分别保存内容，权限一样的文件:

			$> ll /etc/rsyncd.passworda
			-rw------- 1 root root 39 Sep 18 17:12 /etc/rsyncd.passworda

3. 在客户机上分别执行如下命令，即完成文件同步功能:


		$> rsync -azv rsync@ipa::php_env --password-file=/etc/rsyncd.passworda /usr/local/webserver/php/etc/fpm.d/

至此，已完成文件同步的功能。但如果仅仅依靠计划任务来对该目录下的文件进行同步，有些不便捷。所以引入以下工具配合rsync来对配置文件进行同步管理。


#### 2. inotify-tools配置以及使用 ####

1. 在`github`上下载`inotify-tools`源码包地址是`http://github.com/downloads/rvoicilas/inotify-tools/inotify-tools-3.14.tar.gz`


2. 解压并且编译安装


		$> tar zxvf inotify-tools-3.14.tar.gz
		$> cd inotify-tools-3.14
		$>  ./configure 
		$> make
		$> make install


3. 验证是否安装成功


		$> ll /usr/local/bin/inotifywa*
		-rwxr-xr-x 1 root root 44279 Sep 19 09:35 /usr/local/bin/inotifywait
		-rwxr-xr-x 1 root root 41369 Sep 19 09:35 /usr/local/bin/inotifywatch


4. 编写脚本试验功能:

	
	* 脚本内容(监测`/home/guest/sh/a`文件的状态)

			$> cat test.sh 
			#!/bin/bash
			while /usr/local/bin/inotifywait -e modify /home/guest/sh/a ;do
			    echo "file a has been modified"
			done
	
			$> sh test.sh > test_sh.log &
			[1] 29270
			$> Setting up watches.
			Watches established.


	* 测试(对`/home/guest/sh/a`文件进行修改，写入内容，查看日志`test_sh.log`):


			$> echo asdasfd > a 
			$> tail -f test_sh.log 
			/home/guest/sh/a MODIFY 
			file a has been modified

至此，两个工具的特性已经介绍完毕。


#### 3. 功能实现 ####


设想的文件同步功能示意图如下:

![同步功能示意图](http://7xpdnd.com1.z0.glb.clouddn.com/Multi%20Merge%20Process_a.png)

(1). 在客户端推送启动`rsyncd`守护进程所需要的配置文件


		$> ansible all -m copy -a "src=/home/guest/rsync/rsyncd.passworda dest=/etc/" -k -i /home/.houzi/hosts/php.hosts
		$> ansible all -m copy -a "src=/home/guest/rsync/rsyncd.passworda dest=/root/rsync/" -k -i /home/.houzi/hosts/php.hosts


(2). 启动`rsyncd`守护进程


		$> ansible all -m shell -a ". /etc/profile && /usr/bin/rsync --daemon /etc/rsyncd.conf" -k -i /home/.houzi/hosts/php.hosts
		$> ansible all -m shell -a "ps aux |grep rsync | grep -v grep " -k -i /home/.houzi/hosts/php.hosts		


(3). 在服务端上测试同步是否成功:


		$> /usr/bin/rsync -avzP --progress --delete --password-file=/etc/rsyncd.passworda /usr/local/webserver/php/etc/fpm.d/ root@ipa::php_env
		@ERROR: auth failed on module php_env
		rsync error: error starting client-server protocol (code 5) at main.c(1503) [sender=3.0.6]
		$> /usr/bin/rsync -avzP --progress --delete --password-file=/etc/rsyncd.passworda /usr/local/webserver/php/etc/fpm.d/ root@ipb::php_env
		@ERROR: auth failed on module php_env
		rsync error: error starting client-server protocol (code 5) at main.c(1503) [sender=3.0.6]
		$> /usr/bin/rsync -avzP --progress --delete --password-file=/etc/rsyncd.passworda /usr/local/webserver/php/etc/fpm.d/ root@ipc::php_env
		@ERROR: auth failed on module php_env
		rsync error: error starting client-server protocol (code 5) at main.c(1503) [sender=3.0.6]
	

至此，貌似没有一个成功的，遇到这个问题，先检查目标机器上的`/etc/rsync.passwda`文件的权限


		$> ansible all -m shell -a "ls -lthr /etc/rsyn*" -k -i hosts 
		SSH password: 
		192.168.143.171 | SUCCESS | rc=0 >>
		-rw------- 1 root root   15 Aug 17  2015 /etc/rsyncd.password
		-rw------- 1 root root    8 Aug 18  2015 /etc/rsync.password
		-rw------- 1 root root   38 Sep 23 00:34 /etc/rsyncd.passworda
		...
		....太长不写


可以看到`/etc/rsyncd.passworda`该文件权限是`600`,好吧，不是这个原因
想了N久，看了下推下去的hosts，当中包涵了服务端自己本身，导致服务端的`/etc/rsyncd.passwda`认证文件与客户端一样(正确的格式应该是服务端只需要密码，不需要用户名)
删除之后问题解决。

至此，同步服务搭建完成

(4). 推送监测工具`inotify-tools`并解压安装(后期会打成一个rpm包)


(5).脚本推送
		
		#!/bin/bash
		###############################################################################
		#This script is for monitor whether a file or folder is changed,if so , rsync it or reload php-fpm 
		###############################################################################
		
		command=$1
		rsync_File=/usr/local/webserver/php/etc/fpm.d/
		iplist=/root/rsync/iplist
		
		#function rsyncCommand can rsync the pointed file using rsync following the ip in the file defined in var iplist
		function rsyncCommand()
		{
		    while read ip
		    do
		
		        /usr/bin/rsync -avzP --progress --delete --password-file=/etc/rsyncd.passworda ${rsync_File} root@${ip}::php_env
		
		    done < ${iplist}
		
		}
		
		#function minitor is using inotify-tool to look up whether the defined file is changed, with some params to write some format log
		function monitor()
		{
		    /usr/local/bin/inotifywait -mrq --timefmt '%d/%m/%y %H:%M' --format '%T %w%f %e' -e modify,delete,create  $rsync_File | while read Date Time Filename Operation
		    do
		
		        rsyncCommand 
		        if [[ $? -ne 0 ]]; then
		 
		            echo "someting wrong , Please check if rsync-daemon is running in the machines of iplist " >> /tmp/rsync.log
		            echo "------------------------------------------------------------------------------------------------------"  >> /tmp/rsync.log
			
		        else
		
		            echo "${Date} $Time  ${Filename} was ${Operation} & was rsynced to client" >> /tmp/rsync.log
		            echo "---------------------------start to reload php-fpm----------------------------------"  >> /tmp/rsync.log
		            /etc/init.d/php-fpm reload >> /tmp/rsync.log
		            echo "---------------------------php-fpm has been reload----------------------------------"  >> /tmp/rsync.log
		
		        fi
		
		    done
		
		}
		
		function reloadPhpfpm()
		{
		    /usr/local/bin/inotifywait -mrq --timefmt '%d/%m/%y %H:%M' --format '%T %w%f %e' -e modify,delete  $rsync_File |while read Date Time Filename Operation
		
		    do
		
		    	echo "${Date} $Time  ${Filename} has been $Operation, php-fpm is reloading" >> /tmp/reload.log
		        /etc/init.d/php-fpm reload
		
		    done
		}
		case $1 in
			monitor|-M)
				monitor
				;;
		    Rphpfpm|-R)
		        reloadPhpfpm
		        ;;
		    *)
		        echo "Input error,please using monitor|Rphpfpm"
		        exit 1
		        ;;
		esac


(6).启动监测脚本，在客户端上执行一下命令:


		$> /bin/sh /root/rsync/php_env_manager.sh -R >> /root/rsync/php_env_manager.log &


并将其加到开机启动中去。
若出现以下进程，表明自动`reload`脚本部署完成:


		$> ps aux |grep manager
		root      9799  0.0  0.0 106240  1336 pts/1    S    13:57   0:00 /bin/sh /root/rsync/php_env_manager.sh -R
		root      9801  0.0  0.0 106244   824 pts/1    S    13:57   0:00 /bin/sh /root/rsync/php_env_manager.sh -R
		


(7).验证:


* 在`171`上对`/usr/local/webserver/php/etc/fpm.d/juanpi_php_env.conf`文件进行修改(添加一行配置)
* 在`171`上执行`sh /root/rsync/rsync_juanpi.sh`脚本内容如下：
	
	
		$> cat /root/rsync/rsync_juanpi.sh 
		#!/bin/bash
		phpenvFile=/usr/local/webserver/php/etc/fpm.d
		shellFile=/etc/profile.d
		iplist=/root/rsync/iplist
		
		#function rsyncCommand can rsync the pointed file using rsync following the ip in the file defined in var iplist
		function rsyncPhpEnv()
		{
		    while read ip
		    do
		
		        /usr/bin/rsync -avzP --progress --delete --password-file=/etc/rsyncd.passworda ${phpenvFile}/juanpi_php_env.conf root@${ip}::php_env
		
		    done < ${iplist}
		
		}
		
		#function rsyncCommand can rsync the pointed file using rsync following the ip in the file defined in var iplist
		function rsyncshell()
		{
		    while read ip
		    do
		
		        /usr/bin/rsync -avzP --progress --delete --password-file=/etc/rsyncd.passworda ${shellFile}/shell_env.sh root@${ip}::php_shell
		
		    done < ${iplist}
		
		}
		
		case $1 in
		    -P|php|-p)
		    rsyncPhpEnv
		    ;;
		    -S|shell|-s)
		    rsyncshell
		    ;;
		    *|-H|-h|--help)
		    echo "Usage sh $0 -P|php|-p for rsync fpm_env -S|shell|-s for rsync shell_env"
		    exit 1
		esac


* 可以看到在每台客户机上的`/tmp/reload.log`有如下日志输出


		$> tail -f /tmp/reload.log 
		26/09/16 13:58  /usr/local/webserver/php/etc/fpm.d/.juanpi_php_env.conf.1hga3I has been MODIFY, php-fpm is reloading
则表明自动reload脚本部署完成且已经生效了。


8).结果


因为inotify的监测，做不到对单个文件的mtime进行监测，测试过很多次，对单个文件修改，无法触发自动rsync(原因在后文解释)，如果对目录进行检测，由于`fpm.d`目录下会生成临时文件则会造成`php`的`reload`操作过于频繁(因为计划任务的进行，会导致很多临时文件的产生)。


### 三、最终方案


最终使用的同步方法是确认原配置文件正确之后，使用一个批量`rsync`同步脚本，将修改分发到各个客户端。或者使用`ansible`来进行文件同步。配合计划任务(区分功能与回归测试环境的环境变量值)


### 四、总结

由于`inotify－tools`监控机制是基于文件的`inode`指向。而通过`rsync`方法同步文件调用的是系统的`sendfile`对文件进行修改，这种情况造成的后果是原来被`inotify`检测的文件`inode`已经失效了，虽然文件名称依然一样，但此时这个同名文件的`inode`指向已经是一个全新的节点。故就无法检测到对应的事件。


虽然最终没有达到理想的效果，但是对linux的文件系统又有了新的认识。即每个`inode`包含着文件的元信息：

* 文件的字节数
* 文件的所属(`GID`和`UID`)
* 文件的权限(`r`,`w`,`x`)
* 文件的时间戳(`ctime`,`mtime`,`atime`)
* 链接数
* 文件数据的`block`位置

对文件进行修改操作的时候，文件的`inode`并未改变，此时，`inotifywait`还是可以监视文件内容变化的，也就是说直接对文件进行修改能触发同步脚本。

但是通过`rsync`同步文件之后，该文件名虽然还和原来一致，但是此时文件的`inode`值已经改变，故`inotifywait`无法监视文件内容变化，因为它所监视的文件已经不存在了。

虽然做了一次无用功，但是，让自己对`Linux`文件系统有了新的认知(也反映了自己基础的不牢固)

