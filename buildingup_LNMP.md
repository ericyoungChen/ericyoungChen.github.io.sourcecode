title: LNMP服务器环境配置以及搭建
date: 2015-12-17 15:39:45
tags: [LNMP,服务器搭建,Nginx,MySQL,PHP]
---
### 一、简介 ###

Nginx是俄罗斯人编写的十分轻量级的HTTP服务器,Nginx，它的发音为 “engine X”， 是一个高性能的HTTP和反向代理服务器，同时也是一个IMAP/POP3/SMTP 代理服务器．Nginx是由俄罗斯人 Igor Sysoev为俄罗斯访问量第二的 Rambler.ru站点开发的，它已经在该站点运行超过三年了。Igor Sysoev在建立的项目时,使用基于BSD许可。
在高并发连接的情况下，Nginx是Apache服务器不错的替代品。Nginx同时也可以作为7层负载均衡服务器来使用。Nginx + PHP(FastCGI) 可以承受3万以上的并发连接数，相当于同等环境下Apache的10倍。

Nginx的官方维基：[https://www.nginx.com/resources/wiki/](https://www.nginx.com/resources/wiki/)

![nginx](/img/nginx.gif)

<!--more-->
### 二、系统环境准备 ###

1. REHL系列(内核2.6以上)，在此使用的是CentOS6.5 64位最小化安装。
2. 配置好 IP、 DNS、 网关、 主机名
3. 配置防火墙，开放80、3306端口，在防火墙配置文件添加如下代码段：


		$ vim /etc/sysconfig/iptables
		-A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT #这段是原有的，在这之后添加
		-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT #这是新添加的，开放80端口
		-A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT #这里开房3306端口

4. 关闭SELinux


		$ sed -i 's/SELINUX=*/SELINUX=disabled/' /etc/selinux/config
		$ sentenforce 0

### 三、环境约定 ###

1. 软件源代码包存储位置： /usr/local/src
2. 源码编译安装位置： /usr/local/软件名称
3. 数据库文件存储路径： /usr/local/mysql/var


### 四、软件包下载 ###

打开 /usr/local/src 目录 `$ cd /uer/local/src`

1. 下载nginx：[http://nginx.org/download/nginx-1.4.4.tar.gz](http://nginx.org/download/nginx-1.4.4.tar.gz)
2. 下载pcre(支持nginx伪静态)： [http://sourceforge.net/projects/pcre/files/pcre/8.34/pcre-8.34.tar.gz/download](http://sourceforge.net/projects/pcre/files/pcre/8.34/pcre-8.34.tar.gz/download)的此处下载的文件名为download，将其更名为pcre-8.34.tar.gz
3. 下载MySQL:[http://mirrors.sohu.com/mysql/MySQL-5.5/mysql-5.5.45.tar.gz](http://mirrors.sohu.com/mysql/MySQL-5.5/mysql-5.5.45.tar.gz)
4. 下载PHP：[http://cn2.php.net/distributions/php-5.5.7.tar.gz](http://cn2.php.net/distributions/php-5.5.7.tar.gz)
5. 下载cmake(MySQL编译工具):[http://www.cmake.org/files/v2.8/cmake-2.8.12.1.tar.gz](http://www.cmake.org/files/v2.8/cmake-2.8.12.1.tar.gz)
6. 下载libmcrypt(PHPlibmcrypt模块):[http://nchc.dl.sourceforge.net/project/mcrypt/Libmcrypt/2.5.8/libmcrypt-2.5.8.tar.gz](http://nchc.dl.sourceforge.net/project/mcrypt/Libmcrypt/2.5.8/libmcrypt-2.5.8.tar.gz)
7. 	下载GD包(php页面图片验证码支持):[https://github.com/libgd/libgd/archive/GD_2_0_33.tar.gz](https://github.com/libgd/libgd/archive/GD_2_0_33.tar.gz)

### 五、安装编译工具以及库文件 ###

		$ yum groupinstall "Development Tools" -y 
		$ yum groupinstall "Server Platform Development" -y
		$ yum install pcre-devel -y
		$ yum install gcc-c++ -y

### 六、软件安装 ###

#### 1、安装 cmake ####


		$ cd /usr/local/src
		$ tar zxvf 
		$ cmake-2.8.12.1.tar.gz
		$ cd cmake-2.8.8
		$ ./configure --prefix=/usr/local/cmake
		$ make # 编译
		$ make install # 安装
		$ sed -i '$a export PATH=$PATH:/usr/local/cmake/bin' /etc/profile # 在path路径添加cmake执行文件路径
		$ source /etc/profile #使配置生效

#### 2、安装pcre ####

		$ cd /usr/local/src
		$ mkdir /usr/local/pcre #创建安装目录
		$ tar zxvf pcre-8.34.tar.gz
		$ cd pcre-8.34 
		$ ./configure #配置
		$ make && make install #编译以及安装

#### 3、安装 libmcrypt ####

		$ cd /opt/local/src
		$ tar zxvf libmcrypt-2.5.8.tar.gz #解压
		$ cd libmcrypt-2.5.8 #进入目录
		$ ./configure #配置
		$ make && make install

#### 4、安装gd库 ####

		$ cd /opt/local/src
		$ tar zxvf GD_2_0_33.tar.gz
		$ cd libgd-GD_2_0_33/src
		$ ./configure --enable-m4_pattern_allow --prefix=/opt/local/gd --with-jpeg=/usr/lib --with-png=/usr/lib --with-xpm=/usr/lib --with-freetype=/usr/lib --with-fontconfig=/usr/lib #配置
		$ make && make install

#### 5、安装MySQL ####

 1. 编译安装MySQL


		$ groupadd mysql #添加mysql用户组
		$ useradd -g mysql mysql -s /bin/false #创建mysql用户并加入到mysql用户组，不允许mysql用户直接登录系统
		$ mkdir -p /usr/local/mysql/var #创建MySQL目录以及数据库存放目录
		$ chown -R mysql:mysql /usr/local/mysql/var $设置目录权限
		$ cd /usr/local/src
		$ tar zxvf mysql-5.5.45.tar.gz # 解压
		$ cd mysql-5.5.45
		$ cmake . -DMAKE_INSTALL_PREFIX=/usr/local/mysql -DMYSQL_DATADIR=/usr/local/mysql/var -DSYSCONFDIR=/ete # 配置制定编译的软件安装路径、数据库文件路径、配置文件路径 
		# 多次提示未安装Cureses库，移除CMakeCache.txt后`rm CMakeCache.txt`，安装ncurses：`yum install ncurses-devel -y`执行上述语句即可完成配置
		$ make && make install

2. 修改配置文件以及设置启动


		$ cd /usr/loacal/.msyql
		$ cp ./support-files/my-medium.cnf /etc/my.cnf #若提示该目录下已存在文件，直接覆盖即可
		$ vim /etc/my.cnf # 在[mysqld]段增加
		  datadir = /usr/local/mysql/var # 添加MySQL数据库路径
		$ ./scripts/mysql_install_db --user=mysql #初始化数据库
		$ cp ./support-flies/mysql.server /etc/rc.d/init.d/mysqld # 将MySQL加入系统启动
		$ chomd 755 /etc/init.d/mysqld # 增加执行权限
		$ chkconfig mysqld on # 设置开机启动

3. 修改启动脚本


		$ vim /etc/rc.d/init.d/mysqld
		   basedir=/usr/local/mysql # 指定MySQL程序安装路径
		   datadir=/usr/local/mysql/var # 指定MySQL数据库存放路径
		   :wq #保存并退出
		$ service mysqld start # 启动MySQL服务
		$ vim /etc/profile 
		   export PATH=$PATH:/usr/local/mysql/bin
		   :wq # 将MySQL服务添加到环境变量并且保存
		$ source /etc/profile # 使配置生效
		$ mkdir /var/lib/mysql #创建目录
		$ ln -s /tmp/mysql.sock /var/lib/mysql/mysql.sock # 创建软链接

4. 设置密码并且进入mysql控制台


		$ mysql_secure_installation #根据按照提示设置密码
		$ /usr/local/mysql/bin/mysqlanmin -u root -p password "123456"#或者按此方法，但是不建议，因为不全

至此MySQL安装完成

#### 6、安装nginx ####

1. 安装


		$ cd /usr/local/src
		$ groupadd www
		$ useradd -g www www -s /bin/false
		$ tar zxvf nginx-1.4.4.tar.gz
		$ cd nginx-1.4.4
		$ yum install zlib zlib-devel # 安装zlib库
		$ ./configure --prefix=/usr/local/nginx --without-http_memcached_module --user=www --group=www --with-http_stub_status_module --with-openssl=/usr/ --with-pcre=/opt/local/src/pcre-8.34 #--with-pcre注意这里指向的是pcre源码包解压的路径
		$ make && make install

2.设置nginx配置文件以及添加启动脚本


		vim /etc/rc.d/init.d/nginx #编辑启动文件添加下面内容
		
		#!/bin/bash
		# nginx Startup script for the Nginx HTTP Server
		# it is v.0.0.2 version.
		# chkconfig: - 85 15
		# description: Nginx is a high-performance web and proxy server.
		# It has a lot of features, but it's not for everyone.
		# processname: nginx
		# pidfile: /var/run/nginx.pid
		# config: /usr/local/nginx/conf/nginx.conf
		nginxd=/opt/local/nginx/sbin/nginx
		nginx_config=/opt/local/nginx/conf/nginx.conf
		nginx_pid=/opt/local/nginx/logs/nginx.pid
		RETVAL=0
		prog="nginx"
		# Source function library.
		. /etc/rc.d/init.d/functions
		# Source networking configuration.
		. /etc/sysconfig/network
		# Check that networking is up.
		[ ${NETWORKING} = "no" ] && exit 0
		[ -x $nginxd ] || exit 0
		# Start nginx daemons functions.
		start() {
		if [ -e $nginx_pid ];then
		echo "nginx already running...."
		exit 1
		fi
		echo -n $"Starting $prog: "
		daemon $nginxd -c ${nginx_config}
		RETVAL=$?
		echo
		[ $RETVAL = 0 ] && touch /var/lock/subsys/nginx
		return $RETVAL
		}
		# Stop nginx daemons functions.
		stop() {
		echo -n $"Stopping $prog: "
		killproc $nginxd
		RETVAL=$?
		echo
		[ $RETVAL = 0 ] && rm -f /var/lock/subsys/nginx /usr/local/nginx/logs/nginx.pid
		}
		reload() {
		echo -n $"Reloading $prog: "
		#kill -HUP `cat ${nginx_pid}`
		killproc $nginxd -HUP
		RETVAL=$?
		echo
		}
		# See how we were called.
		case "$1" in
		start)
		start
		;;
		stop)
		stop
		;;
		reload)
		reload
		;;
		restart)
		stop
		start
		;;
		status)
		status $prog
		RETVAL=$?
		;;
		*)
		echo $"Usage: $prog {start|stop|restart|reload|status|help}"
		exit 1
		esac
		exit $RETVAL
		#至此，保存并且退出
		:wq
		$ chmod 775 /etc/rc.d/init.d/nginx #添加权限
		$ chkconfig nginx on 
		$ /etc/rc.d/init.d/nginx restart #重启nginx
		$ sevice nginx restart

#### 7、安装PHP ####

1. 编译安装
		

		$ cd /usr/local/src
		$ tar -zvxf php-5.5.7.tar.gz
		$ cd php-5.5.7.
		$ ./configure --prefix=/usr/local/php5 --with-config-file-path=/opt/local/php5/etc --with-mysql=/opt/local/mysql --with-mysql-sock=/tmp/mysql.sock --with-gd --with-iconv --with-zlib --enable-xml --enable-bcmath --enable-shmop --enable-sysvsem --enable-inline-optimization --enable-mbregex --enable-fpm --enable-mbstring --enable-ftp --enable-gd-native-ttf --with-openssl --enable-pcntl --enable-sockets --with-xmlrpc --enable-zip --enable-soap --without-pear --with-gettext --enable-session --with-mcrypt --with-curl --with-jpeg-dir --with-freetype-dir    #配置
		#显示配置失败，依旧多次报错，缺少很多库文件和依赖。每配置一次，报错一次，然后就安装一次库文件和依赖（应该在第五步的时候指令输错了），以下安装 mysql和nginx 也是如此，在此耽误了很长时间
		$ yum -y install libxml2 libxml2-devel
		$ yum -y install openssl openssl-devel
		$ make && make install 

2. 配置启动文件
	

		$ cp /usr/local/src/php-5.5.7/sapi/fpm/init.d.php-fpm /etc/rc.d/init.d/php-fpm #拷贝 php-fpm到启动目录
		$ chmod +x /etc/rc.d/init.d/php-fpm #添加执行权限
		$ chkconfig php-fpm on #设置开机启动

3. 修改配置文件


		$ vim /usr/local/php5/etc/php.ini #编辑配置文件
	
找到：`disable_functions =`
修改为：
		

		disable_functions = passthru,exec,system,chroot,scandir,chgrp, chown,shell_exec,proc_open,proc_get_status,ini_alter,ini_alter,ini_restore,dl,openlog,syslog, readlink,symlink ,popepassthru,stream_socket_server,escapeshellcmd,dll,popen,disk_free_space,checkdnsrr,checkdnsrr, getservbyname,getservbyport ,disk_total_space,posix_ctermid,posix_get_last_error,posix_getcwd, posix_getegid,posix_geteuid,posix_getgid, posix_getgrgid,posix_getgrnam,posix_getgroups,posix_getlogin,posix_getpgid,posix_getpgrp,posix_getpid, posix_getppid,posix_getpwnam,posix_getpwuid, posix_getrlimit, posix_getsid,posix_getuid,posix_isatty, posix_kill,posix_mkfifo,posix_setegid,posix_seteuid,posix_setgid, posix_setpgid,posix_setsid,posix_setuid,posix_strerror,posix_times,posix_ttyname,posix_uname
		#列出 PHP可以禁用的函数，如果某些程序需要用到这个函数，可以删除，取消禁用
		找到：;date.timezone =
		修改为：date.timezone = PRC #设置时区
		找到：expose_php = On
		修改为：expose_php = OFF #禁止显示 php版本的信息
		找到：short_open_tag = Off
		修改为：short_open_tag = ON #支持 php短标签

#### 八、配置nginx支持PHP ####

修改/usr/local/nginx/conf/nginx.conf 配置文件, 需做如下修改


		vi /usr/local/nginx/conf/nginx.conf		
		user www www; #首行user去掉注释 ,修改Nginx运行组为 www www；必须与/usr/local/php/etc/php-fpm.conf中的 user,group配置相同，否则php运行出错
		user www www;
		worker_processes 1;
		events {
		worker_connections 1024;
		}
		http {
		include mime.types;
		default_type application/octet-stream;
		sendfile on;
		keepalive_timeout 65;
		server {
		listen 80;
		server_name localhost;
		location / {
		root html;
		index index .php index.html index.htm;
		}
		location ~ \.php$ {
		root html;
		fastcgi_pass 127.0.0.1:9000;
		fastcgi_index index.php;
		fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
		include fastcgi_params;
		}
		}
		}
		$ /etc/init.d/nginx restart #重启nginx

#### 九、测试篇 ####	


		$ cd /opt/local/nginx/html/ #进入 nginx默认网站根目录
		$ rm -rf /opt/local/nginx/html/* #删除默认测试页
		vim index.php #新建 index.php文件，内容如下：
			<?
			phpinfo();
			?>
		:wq! #保存退出
		$ chown www.www /opt/local /nginx/html/ -R #设置目录所有者
		$ chmod 700 /opt/local /nginx/html/ -R #设置目录权限

在浏览器上输入地址即可显示PHP的信息


			