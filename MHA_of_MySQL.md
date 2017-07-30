title: MySQL高可用架构MHA
date: 2015-12-27 00:04:43
tags: [MySQL,MHA]

---

### 一、关于MHA ###


1. 简介：


MHA(Master High Avaliablity)是目前在MySQL高可用方面一个相对成熟的解决方案，它由日本DeNA公司的youshimaton(现就职于F啊侧book公司)开发，是一套作为MySQL高可用环境下故障切换和主从提升的优秀高可用软件。在MySQL故障切换过程中，MHA能做到短时间内完成数据库故障自动切换操作，且在切换过程中能很大程度上保持数据的一致性，以达到真正意义上的高可用。

2. 结构：

该软件由两部分构成：MHA-Manager(管理节点)和MHA-Node(数据节点)。MHA-Manager可以单独部署在一台独立的机器管理多个Master-Slave(主从)集群，也可以部署在其中一台Slave节点上。MHA-Node运行在每台MySQL服务器上，MHA-Manager会定时探测集群中的Master节点，当Master出现故障时，它可以自动地将最新数据的Slave提升为新的Master，然后将所有其他的Slave重新指向新的Master。整个故障转移过程对应用程序完全透明。 

目前MHA主要支持一主多从的架构，**要搭建MHA，要求一个复制集群中至少要有三台数据库服务器**，一主二从，即一台充当Master，剩下两台充当Slave(其中一个为备用Master)。

架构图如下：

![MHA架构图](http://7xpdnd.com1.z0.glb.clouddn.com/MHA%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

<!--more-->

3. 原理总结：

	* 从宕机或者崩溃的Master保存二进制事件(binlog events);
	* 识别含有最近更新的Slave；
	* 讲差异的中继日志(relay log)应用到其他的Slave；
	* 应用从Master保存的二进制日志事件(binlog events);
	* 将Slave(即备用Master)提升为新的Master；
	* 将其他的Slave指向新的Master

官方介绍： [https://code.google.com/p/mysql-master-ha/](https://code.google.com/p/mysql-master-ha/)

### 二、系统约定 ###

1. 系统以及软件：Centos6.5 x64 最小化安装+MySQL-5.5.45
2. 环境：


	IP地址|角色
	--- | ---
	192.168.1.124|MHA Manager
	192.168.1.126|MySQL Master
	192.168.1.127|MySQL Slave1(备用Master)
	192.168.1.128|MySQL Slave2


### 三、MySQL集群搭建 ###

(1). 配置hosts本地解析

	4台机子都配置相同的hosts解析，内容如下：
		

		$ cat /etc/hosts
			127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
			::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
			192.168.1.124   manager
			192.168.1.126   master
			192.168.1.127   slave1
			192.168.1.128   slave2

	每台机均执行`yum -y install openssh*`同时开启3306端口

(2). 配置四台机器之间ssh免秘钥登录

Manager:


		$ ssh-keygen
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@master
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@slave1
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@slave2

Master:


		$ ssh-keygen
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@manager
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@slave1
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@slave2

Slaver1:


		$ ssh-keygen
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@master
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@manager
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@slave2

Slaver1:


		$ ssh-keygen
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@master
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@manager
		$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@slave1




(3). 安装MySQL

	(1) 在Master,Slave1,Slave2上安装MySQL服务，在此安装的是MySQL-5.5.45.tar.gz,为节约时间，在此使用脚本安装，脚本内容如下：


		$ cat mysql_install.sh
			#!/bin/bash
  
		DATADIR='/data/mysql/data'
		VERSION='mysql-5.5.45'
		export LANG=zh_CN.UTF-8
  
		#Source function library.
		. /etc/init.d/functions
  
		#camke install mysql5.5.X
		install_mysql(){
        read -p "please input a password for root: " PASSWD
        if [ ! -d $DATADIR ];then
                mkdir -p $DATADIR
        fi
        yum install cmake make gcc-c++ bison-devel ncurses-devel -y
        id mysql &>/dev/null
        if [ $? -ne 0 ];then
                useradd mysql -s /sbin/nologin -M
        fi
        useradd mysql -s /sbin/nologin -M
        #change datadir owner to mysql
        chown -R mysql.mysql $DATADIR
        cd
        wget http://mirrors.sohu.com/mysql/MySQL-5.5/mysql-5.5.45.tar.gz
        tar xf $VERSION.tar.gz
        cd $VERSION
        cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/$VERSION \
        -DMYSQL_DATADIR=$DATADIR \
        -DMYSQL_UNIX_ADDR=$DATADIR/mysql.sock \
        -DDEFAULT_CHARSET=utf8 \
        -DDEFAULT_COLLATION=utf8_general_ci \
        -DENABLED_LOCAL_INFILE=ON \
        -DWITH_INNOBASE_STORAGE_ENGINE=1 \
        -DWITH_FEDERATED_STORAGE_ENGINE=1 \
        -DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
        -DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 \
        -DWITHOUT_PARTITION_STORAGE_ENGINE=1
        make && make install
        if [ $? -ne 0 ];then
                action "install mysql is failed"  /bin/false
                exit $?
        fi
        sleep 2
        #link
        ln -s /usr/local/$VERSION/ /usr/local/mysql
        ln -s /usr/local/mysql/bin/* /usr/bin/
        #copy config and start file
        /bin/cp /usr/local/mysql/support-files/my-small.cnf /etc/my.cnf
        cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
        chmod 700 /etc/init.d/mysqld
        #init mysql
        /usr/local/mysql/scripts/mysql_install_db  --basedir=/usr/local/mysql --datadir=$DATADIR --user=mysql
        if [ $? -ne 0 ];then
                action "install mysql is failed"  /bin/false
                exit $?
        fi
        #check mysql
        /etc/init.d/mysqld start
        if [ $? -ne 0 ];then
                action "mysql start is failed"  /bin/false
                exit $?
        fi
        chkconfig --add mysqld
        chkconfig mysqld on
        /usr/local/mysql/bin/mysql -e "update mysql.user set password=password('$PASSWD') where host='localhost' and user='root';"
        /usr/local/mysql/bin/mysql -e "update mysql.user set password=password('$PASSWD') where host='127.0.0.1' and user='root';"
        /usr/local/mysql/bin/mysql -e "delete from mysql.user where password='';"
        /usr/local/mysql/bin/mysql -e "flush privileges;"
        #/usr/local/mysql/bin/mysql -e "select version();" >/dev/null 2>&1
        if [ $? -eq 0 ];then
                echo "+---------------------------+"
                echo "+------mysql安装完成--------+"
                echo "+---------------------------+"
        fi
        #/etc/init.d/mysqld stop
		}
		install_mysql

	(2)建立Master,Slave1,Slave2之间的主从复制，修改三台机器的server-id，确保是唯一的，并且开启log-bin选项。

		
		#在Master下：
		$ egrep "log-bin|server-id" /etc/my.cnf
		server-id = 1
		log-bin=mysql-bin

		#在Slave1下：
		$ egrep "log-bin|server-id" /etc/my.cnf
		server-id = 51
		log-bin=mysql-bin
		#在Slave2下：
		$ egrep "log-bin|server-id" /etc/my.cnf
		server-id = 52
		log-bin=mysql-bin

	(3)在Master,Slave1上配置主从同步的帐号，因为Slave1是备用的Master，所以这台机器也需要进行授权。


		$ mysql -u root -p; # 进入控制台
			mysql> grant all privileges on *.* to 'rep'@'192.168.1.%' identified by 'rep123';
			Qurey OK,0 rows affected (0.00 sec)

			mysql> flush privileges;
			Query OK, 0 rows affected (0.01 sec)

		
	查看Master的状态信息。在Master上执行下述语句：
		
	
		mysql> show master status;
		+------------------+----------+--------------+------------------+
		| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
		+------------------+----------+--------------+------------------+
		| mysql-bin.000002 |      107 |              |                  |
		+------------------+----------+--------------+------------------+
		1 row in set (0.00 sec)

		#记住此时的File和Position值

		
	在Slave1，Slave2上均执行主从同步


		mysql> change master to master_host='192.168.1.126', master_port=3306,master_user='rep',master_password='rep123',master_log_file='mysql-bin.000002',master_log_pos=107;
		Query OK, 0 rows affected (0.02 sec)
		#查看主从同步的状态
		mysql> show slave status\G;
			*************************** 1. row ***************************
               		Slave_IO_State: Waiting for master to send event
                  		Master_Host: 192.168.1.126
                  		Master_User: rep
                  		Master_Port: 3306
                	  Connect_Retry: 60
              		Master_Log_File: mysql-bin.000002
          		Read_Master_Log_Pos: 107
               	 	 Relay_Log_File: slave1-relay-bin.000002
                  	  Relay_Log_Pos: 253
          	  Relay_Master_Log_File: mysql-bin.000002
               	   Slave_IO_Running: Yes  			#当此行和下一行显示的Yes表示主从同步设置成功
              	  Slave_SQL_Running: Yes
              		Replicate_Do_DB: 
         		Replicate_Ignore_DB: 
           	 	 Replicate_Do_Table: 
       	 	 Replicate_Ignore_Table: 
      		Replicate_Wild_Do_Table: 
		Replicate_Wild_Ignore_Table:
                   		 Last_Errno: 0
                   		 Last_Error: 
                 	   Skip_Counter: 0
          		Exec_Master_Log_Pos: 107
              		Relay_Log_Space: 410
              		Until_Condition: None
               		 Until_Log_File: 
                	  Until_Log_Pos: 0
           		 Master_SSL_Allowed: No
           		 Master_SSL_CA_File: 
           		 Master_SSL_CA_Path: 
              		Master_SSL_Cert: 
            	  Master_SSL_Cipher: 
              	 	 Master_SSL_Key: 
        	  Seconds_Behind_Master: 0
		Master_SSL_Verify_Server_Cert: No
                	  Last_IO_Errno: 0
              		  Last_IO_Error: 
               		 Last_SQL_Errno: 0
               		 Last_SQL_Error: 
		Replicate_Ignore_Server_Ids: 
             	   Master_Server_Id: 1
		1 row in set (0.00 sec)

		ERROR: 
		No query specified
(4)两台Slave服务器设置read_only(从库对外提供读服务，之所以没有写进配置文件，是因为Slave随时会提升为Master)


		#设置Slave1
		[root@slave1 ~]# mysql -uroot -p215713 -e "set global read_only=1"
		#设置Slave2
		[root@slave2 ~]# mysql -uroot -p215713 -e "set global read_only=1"
		
至此，MySQL一主二从集群搭建完成。

### 四、MHA部署 ###

1. 创建MHA管理用的复制帐号


		#需要在每台数据库上都要创建4个帐号
		mysql> grant all privileges on *.* to 'mha_rep'@'192.168.1.124' identified by '123456';
		Query OK, 0 rows affected (0.00 sec)
 
		mysql> grant all privileges on *.* to 'mha_rep'@'192.168.1.126' identified by '123456';
		Query OK, 0 rows affected (0.00 sec)
 
		mysql> grant all privileges on *.* to 'mha_rep'@'192.168.1.127' identified by '123456';
		Query OK, 0 rows affected (0.00 sec)
 
		mysql> grant all privileges on *.* to 'mha_rep'@'192.168.1.128' identified by '123456';
		Query OK, 0 rows affected (0.00 sec)
 
		mysql> flush privileges;
		Query OK, 0 rows affected (0.00 sec)

2. 下载软件源码包mha4mysql-node,mha4mysql-manager地址如下，自备梯子： [https://code.google.com/p/mysql-master-ha/wiki/Downloads?tm=2](https://code.google.com/p/mysql-master-ha/wiki/Downloads?tm=2)


3. 在所有节点安装mha4mysql-node


		# 安装基础软件包
		$ yum install gcc perl-DBD-MySQL perl-CPAN perl-ExtUtils-CBuilder perl-ExtUtils-MakeMaker -y
		$ tar zxvf mha4mysql-node-0.56.tar.gz
		$ cd mha4mysql-node-0.56
		$ perl Makefile.PL
		$ make & make install


4. 在Manager上安装mha4mysql-manager


		# 安装基础软件包
		$  yum install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager perl-Time-HiRes -y
		$ tar zxvf mha4mysql-manager-0.56.tar.gz
		$ cd mha4mysql-manager-0.56
		$ perl Makefile.PL
		$ make && make install
		#复制相关文件
		$ mkdir /etc/masterha
		$ mkdir -p /master/app1
		$ cp samples/conf/* /etc/masterha/
		$ cp samples/scripts/* /scripts
		

5.修改配置文件

		
		$ cat /etc/masterha/app1.cnf
		[server default]
		user=mha_rep							#MHA管理MySQL的用户名
		password=123456							#MHA管理MySQL的密码
		manager_workdir=/masterha/app1			#MHA的工作目录
		manager_log=/masterha/app1/manager.log	#MHA的日志路径
		ssh_user=root							#免秘钥登陆的用户名
		repl_user=rep							#主从复制账号，用来在主从之间同步数据
		repl_password=rep123
		ping_interval=1							#ping间隔时间，用来检查master是否正常

		[server1]
		hostname=192.168.1.126
		master_binlog_dir=/data/mysql/data/
		candidate_master=1						#master宕机后，优先启用这台作为master

		[server2]
		hostname=192.168.1.127
		master_binlog_dir=/data/mysql/data/
		candidate_master=1

		[server3]
		hostname=192.168.1.128
		master_binlog_dir=/data/mysql/data/
		no_master=1								 #设置no_master=1，使服务器不能成为master
	

**设置relay log的清除方式(在每个slave节点上)**


		#在slave1操作：
		[root@slave1 ~]# mysql -uroot -p215713 -e "set global relay_log_purge=0"
		#在slave2操作：
		[root@slave2 ~]# mysql -uroot -p215713 -e "set global relay_log_purge=0"
注意：
MHA在发生切换的过程中，从库的恢复过程中依赖于relay log的相关信息，所以这里要将relay log的自动清除设置为OFF，采用手动清除relay log的方式。在默认情况下，从服务器上的中继日志会在SQL线程执行完毕后被自动删除。但是在MHA环境中，这些中继日志在恢复其他从服务器时可能会被用到，因此需要禁用中继日志的自动删除功能。定期清除中继日志需要考虑到复制延时的问题。在ext3的文件系统下，删除大的文件需要一定的时间，会导致严重的复制延时。为了避免复制延时，需要暂时为中继日志创建硬链接，因为在linux系统中通过硬链接删除大文件速度会很快。（在mysql数据库中，删除大表时，通常也采用建立硬链接的方式）


### 五、验证搭建效果 ###

1. 检查ssh相互通信是否成功


		[root@manager masterha]# masterha_check_ssh --conf=/etc/masterha/app1.cnf 
		Sun Dec 27 20:45:24 2015 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
		Sun Dec 27 20:45:24 2015 - [info] Reading application default configuration from /etc/masterha/app1.cnf..
		Sun Dec 27 20:45:24 2015 - [info] Reading server configuration from /etc/masterha/app1.cnf..
		Sun Dec 27 20:45:24 2015 - [info] Starting SSH connection tests..
		Sun Dec 27 20:45:26 2015 - [debug] 
		Sun Dec 27 20:45:24 2015 - [debug]  Connecting via SSH from root@192.168.1.126(192.168.1.126:22) to root@192.168.1.127(192.168.1.127:22)..
		Sun Dec 27 20:45:25 2015 - [debug]   ok.
		Sun Dec 27 20:45:25 2015 - [debug]  Connecting via SSH from root@192.168.1.126(192.168.1.126:22) to root@192.168.1.128(192.168.1.128:22)..
		Sun Dec 27 20:45:26 2015 - [debug]   ok.
		Sun Dec 27 20:45:27 2015 - [debug] 
		Sun Dec 27 20:45:25 2015 - [debug]  Connecting via SSH from root@192.168.1.127(192.168.1.127:22) to root@192.168.1.126(192.168.1.126:22)..
		Sun Dec 27 20:45:26 2015 - [debug]   ok.
		Sun Dec 27 20:45:26 2015 - [debug]  Connecting via SSH from root@192.168.1.127(192.168.1.127:22) to root@192.168.1.128(192.168.1.128:22)..
		Sun Dec 27 20:45:26 2015 - [debug]   ok.
		Sun Dec 27 20:45:27 2015 - [debug] 
		Sun Dec 27 20:45:25 2015 - [debug]  Connecting via SSH from root@192.168.1.128(192.168.1.128:22) to root@192.168.1.126(192.168.1.126:22)..
		Sun Dec 27 20:45:26 2015 - [debug]   ok.
		Sun Dec 27 20:45:26 2015 - [debug]  Connecting via SSH from root@192.168.1.128(192.168.1.128:22) to root@192.168.1.127(192.168.1.127:22)..
		Sun Dec 27 20:45:27 2015 - [debug]   ok.
		Sun Dec 27 20:45:27 2015 - [info] All SSH connection tests passed successfully.



2. 用masterha_check_repl工具检查MySQL主从复制是否成功


		[root@manager masterha]# masterha_check_repl --conf=/etc/masterha/app1.cnf 
		Sun Dec 27 20:46:28 2015 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
		Sun Dec 27 20:46:28 2015 - [info] Reading application default configuration from /etc/masterha/app1.cnf..
		Sun Dec 27 20:46:28 2015 - [info] Reading server configuration from /etc/masterha/app1.cnf..
		Sun Dec 27 20:46:28 2015 - [info] MHA::MasterMonitor version 0.56.
		Sun Dec 27 20:46:29 2015 - [info] Multi-master configuration is detected. Current primary(writable) master is 192.168.1.126(192.168.1.126:3306)
		Sun Dec 27 20:46:29 2015 - [info] Master configurations are as below: 
		Master 192.168.1.127(192.168.1.127:3306), replicating from 192.168.1.126(192.168.1.126:3306), read-only
		Master 192.168.1.126(192.168.1.126:3306), replicating from 192.168.1.127(192.168.1.127:3306)

		Sun Dec 27 20:46:29 2015 - [info] GTID failover mode = 0
		Sun Dec 27 20:46:29 2015 - [info] Dead Servers:
		Sun Dec 27 20:46:29 2015 - [info] Alive Servers:
		Sun Dec 27 20:46:29 2015 - [info]   192.168.1.126(192.168.1.126:3306)
		Sun Dec 27 20:46:29 2015 - [info]   192.168.1.127(192.168.1.127:3306)
		Sun Dec 27 20:46:29 2015 - [info]   192.168.1.128(192.168.1.128:3306)
		Sun Dec 27 20:46:29 2015 - [info] Alive Slaves:
		Sun Dec 27 20:46:29 2015 - [info]   192.168.1.127(192.168.1.127:3306)  Version=5.5.45-log (oldest major version between slaves) log-bin:enabled
		Sun Dec 27 20:46:29 2015 - [info]     Replicating from 192.168.1.126(192.168.1.126:3306)
		Sun Dec 27 20:46:29 2015 - [info]     Primary candidate for the new Master (candidate_master is set)
		Sun Dec 27 20:46:29 2015 - [info]   192.168.1.128(192.168.1.128:3306)  Version=5.5.45-log (oldest major version between slaves) log-bin:enabled
		Sun Dec 27 20:46:29 2015 - [info]     Replicating from 192.168.1.126(192.168.1.126:3306)
		Sun Dec 27 20:46:29 2015 - [info]     Not candidate for the new Master (no_master is set)
		Sun Dec 27 20:46:29 2015 - [info] Current Alive Master: 192.168.1.126(192.168.1.126:3306)
		Sun Dec 27 20:46:29 2015 - [info] Checking slave configurations..
		Sun Dec 27 20:46:29 2015 - [info] Checking replication filtering settings..
		Sun Dec 27 20:46:29 2015 - [info]  binlog_do_db= , binlog_ignore_db= 
		Sun Dec 27 20:46:29 2015 - [info]  Replication filtering check ok.
		Sun Dec 27 20:46:29 2015 - [info] GTID (with auto-pos) is not supported
		Sun Dec 27 20:46:29 2015 - [info] Starting SSH connection tests..
		Sun Dec 27 20:46:32 2015 - [info] All SSH connection tests passed successfully.
		Sun Dec 27 20:46:32 2015 - [info] Checking MHA Node version..
		Sun Dec 27 20:46:33 2015 - [info]  Version check ok.
		Sun Dec 27 20:46:33 2015 - [info] Checking SSH publickey authentication settings on the current master..
		Sun Dec 27 20:46:33 2015 - [info] HealthCheck: SSH to 192.168.1.126 is reachable.
		Sun Dec 27 20:46:33 2015 - [info] Master MHA Node version is 0.56.
		Sun Dec 27 20:46:33 2015 - [info] Checking recovery script configurations on 192.168.1.126(192.168.1.126:3306)..
		Sun Dec 27 20:46:33 2015 - [info]   Executing command: save_binary_logs --command=test --start_pos=4 --binlog_dir=/data/mysql/data/ --output_file=/var/tmp/save_binary_logs_test --manager_version=0.56 --start_file=mysql-bin.000002 
		Sun Dec 27 20:46:33 2015 - [info]   Connecting to root@192.168.1.126(192.168.1.126:22).. 
		  Creating /var/tmp if not exists..    ok.
		  Checking output directory is accessible or not..
		   ok.
		  Binlog found at /data/mysql/data/, up to mysql-bin.000002
		Sun Dec 27 20:46:33 2015 - [info] Binlog setting check done.
		Sun Dec 27 20:46:33 2015 - [info] Checking SSH publickey authentication and checking recovery script configurations on all alive slave servers..
		Sun Dec 27 20:46:33 2015 - [info]   Executing command : apply_diff_relay_logs --command=test --slave_user='mha_rep' --slave_host=192.168.1.127 --slave_ip=192.168.1.127 --slave_port=3306 --workdir=/var/tmp --target_version=5.5.45-log --manager_version=0.56 --relay_log_info=/data/mysql/data/relay-log.info  --relay_dir=/data/mysql/data/  --slave_pass=xxx
		Sun Dec 27 20:46:33 2015 - [info]   Connecting to root@192.168.1.127(192.168.1.127:22).. 
		  Checking slave recovery environment settings..
		    Opening /data/mysql/data/relay-log.info ... ok.
		    Relay log found at /data/mysql/data, up to slave1-relay-bin.000002
		    Temporary relay log file is /data/mysql/data/slave1-relay-bin.000002
		    Testing mysql connection and privileges.. done.
		    Testing mysqlbinlog output.. done.
		    Cleaning up test file(s).. done.
		Sun Dec 27 20:46:34 2015 - [info]   Executing command : apply_diff_relay_logs --command=test --slave_user='mha_rep' --slave_host=192.168.1.128 --slave_ip=192.168.1.128 --slave_port=3306 --workdir=/var/tmp --target_version=5.5.45-log --manager_version=0.56 --relay_log_info=/data/mysql/data/relay-log.info  --relay_dir=/data/mysql/data/  --slave_pass=xxx
		Sun Dec 27 20:46:34 2015 - [info]   Connecting to root@192.168.1.128(192.168.1.128:22).. 
		  Checking slave recovery environment settings..
		    Opening /data/mysql/data/relay-log.info ... ok.
		    Relay log found at /data/mysql/data, up to slave2-relay-bin.000002
		    Temporary relay log file is /data/mysql/data/slave2-relay-bin.000002
		    Testing mysql connection and privileges.. done.
		    Testing mysqlbinlog output.. done.
		    Cleaning up test file(s).. done.
		Sun Dec 27 20:46:34 2015 - [info] Slaves settings check done.
		Sun Dec 27 20:46:34 2015 - [info] 
		192.168.1.126(192.168.1.126:3306) (current master)
		 +--192.168.1.127(192.168.1.127:3306)
		 +--192.168.1.128(192.168.1.128:3306)

		Sun Dec 27 20:46:34 2015 - [info] Checking replication health on 192.168.1.127..
		Sun Dec 27 20:46:34 2015 - [info]  ok.
		Sun Dec 27 20:46:34 2015 - [info] Checking replication health on 192.168.1.128..
		Sun Dec 27 20:46:34 2015 - [info]  ok.
		Sun Dec 27 20:46:34 2015 - [warning] master_ip_failover_script is not defined.
		Sun Dec 27 20:46:34 2015 - [warning] shutdown_script is not defined.
		Sun Dec 27 20:46:34 2015 - [info] Got exit code 0 (Not master dead).

		MySQL Replication Health is OK.


3. 启动MHA manager，并监控日志文件


		[root@manager masterha]# nohup masterha_manager --conf=/etc/masterha/app1.cnf > /tmp/mha_manager.log 2>&1 &
		[1] 4037


4. 测试Master宕机后，是否自动切换Master


		#测试前先查看Slave1，Slave2的主从情况
		#Slave1
		[root@slave1 mysql]# mysql -u root -p215713 -h 127.0.0.1 -e 'show slave status\G' | grep Yes
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
		#Slave2
		[root@slave2 ~]# mysql -u root -p215713 -h 127.0.0.1 -e 'show slave status\G' | grep Yes
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
		#停止Master的MySQL服务
		[root@master ~]# service mysqld stop
		Shutting down MySQL. SUCCESS!
		#查看manager的日志文件
		
		----- Failover Report -----

		app1: MySQL Master failover 192.168.1.126(192.168.1.126:3306) to 192.168.1.127(192.168.1.127:3306)

		Master 192.168.1.126(192.168.1.126:3306) is down!

		Check MHA Manager logs at manager:/masterha/app1/manager.log for details.

		Started automated(non-interactive) failover.
		The latest slave 192.168.1.127(192.168.1.127:3306) has all relay logs for recovery.
		Selected 192.168.1.127(192.168.1.127:3306) as a new master.
		192.168.1.127(192.168.1.127:3306): OK: Applying all logs succeeded.
		192.168.1.128(192.168.1.128:3306): This host has the latest relay log events.
		Generating relay diff files from the latest slave succeeded.
		192.168.1.128(192.168.1.128:3306): OK: Applying all logs succeeded. Slave started, replicating from 192.168.1.53(192.168.1.53:3306)
		Master failover to 192.168.1.127(192.168.1.127:3306) done, but recovery on slave partially failed.
		#查看Slave2的slave状态
		mysql> show slave status\G;
		*************************** 1. row ***************************
               		Slave_IO_State: Waiting for master to send event
                  		Master_Host: 192.168.1.127
                  		Master_User: rep
                  		Master_Port: 3306
                		Connect_Retry: 60
              		Master_Log_File: mysql-bin.000002
          		Read_Master_Log_Pos: 798
               		Relay_Log_File: slave2-relay-bin.000002
                		Relay_Log_Pos: 253
        		Relay_Master_Log_File: mysql-bin.000002
             		Slave_IO_Running: Yes
            		Slave_SQL_Running: Yes
              		Replicate_Do_DB: 
          		Replicate_Ignore_DB: 
           		Replicate_Do_Table: 
       		Replicate_Ignore_Table: 
      		Replicate_Wild_Do_Table: 
		Replicate_Wild_Ignore_Table: 
                   		Last_Errno: 0
                   		Last_Error: 
                 		Skip_Counter: 0
          		Exec_Master_Log_Pos: 798
              		Relay_Log_Space: 410
              		Until_Condition: None
               		Until_Log_File: 
                		Until_Log_Pos: 0
          		 Master_SSL_Allowed: No
           		Master_SSL_CA_File: 
           		Master_SSL_CA_Path: 
              		Master_SSL_Cert: 
            		Master_SSL_Cipher: 
               		Master_SSL_Key: 
        		Seconds_Behind_Master: 0
		Master_SSL_Verify_Server_Cert: No
                		Last_IO_Errno: 0
                		Last_IO_Error: 
               		Last_SQL_Errno: 0
               		Last_SQL_Error: 
		Replicate_Ignore_Server_Ids: 
             	Master_Server_Id: 52
		1 row in set (0.00 sec)

		ERROR: 
		No query specified
可以看到Slave2指向的Master是Slave1
至此完成故障时转移Master至Slave1。
 
		
