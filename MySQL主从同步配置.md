title: MySQL主从同步配置
date: 2015-12-18 11:12:02
tags: [MySQL,数据库]
---
>前言：MySQL的主从同步是构建MySQL高性能高可用架构的基础，关于MySQL的安装和配置，在前一篇博文已经描述的很详细，在此不做赘述。本文重点在于MySQL的主从同步配置。

### 一、原理概述 ###

MySQL数据的复制方式是安装有该服务的某台主机(master)中的日志文件(binlog)复制到其他主机(slave)上，slave读取对比位于本机的relaylog，并执行一遍所需要的复制来实现的。

复制过程中，一个服务器充当主服务器(master)，而一个或者多个服务器其他服务器充当从服务器。主服务器将更新写入二进制日志文件(binlog)，并维护文件的一个索引以便于跟踪日志循环。而这些日志可以被从服务器读取并且与从服务器自己的日志文件(relaylog)，进行对比，找到最近一次更新的位置，进行写入，完成后，从服务器才开始激活一个新的SQL线程进行数据的读写操作。

### 二、工作描述 ###

1. master将数据的改变记录到二进制日志(binary log)中；
2. slave将master的binary log中的事件内容拷贝到它的中继日志(relay log)；
3. slave重做relay log中的事件，将改变它自己的数据

![原理图示](http://7xpaqv.com1.z0.glb.clouddn.com/Rep.jpg)

<!--more-->

### 三、配置 ###

约定：

 * 主库ip:192.168.137.226
 * 从库ip:192.168.137.179

> MySQL的安装和基本的配置上一篇博文中描写的比较详细，在此不做赘述。详情可以[戳我]()


1. 配置MySQL主服务器(192.168.137.226)


		$ mysql -u root -p #进入MySQL控制台
			>create database youngdb; #建立数据库youngdb
			>insert into mysql.user(Host,User,Password) values('localhost','young',password('123321')); #创建用户young，密码123321
			>grant all on youngdb.* to 'young'@'192.168.137.179' identified by '123321' with grant option; #授权用户young从192.168.137.179(即从库所在的ip地址)完全访问数据库db1
			>insert into mysql.user(Host,User,Password) values('localhost','youngdbbak',password('123321')); #建立MySQL主从数据库同步用户youngdbbak密码123321
			>flush privileges; #刷新系统授权表
			>grant replication slave on *.* to 'youngdbbak'@'192.168.137.179' identified by '123321' with grant option; #授权用户youngdbbak只能从192.168.137.179这个IP访问主服务器192.168.137.226上面的数据库，并且只具有数据库备份的权限
2. 把MySQL主服务器192.168.137.226中的数据库youngdb导入到MySQL从服务器192.168.137.179中

	* 导出数据库youngdb
			

			$ mysqldump -u root -p --default-character-set=utf8 --opt -Q -R --skip-lock-tables youngdb > /home/youngdbbak.sql #在MySQL主服务器进行操作，导出数据库youngdb到/home/youngdbbak.sql
			$ scp /home/youngdbbak.sql root@192.168.137.179:/home #把home目录下的youngdbbak.sql 数据库文件上传到MySQL从服务器的home目录下面
	备注：在导出之前可以先进入MySQL控制台执行下面命令

			mysql -u root -p;
				>flush tables with read lock; #数据库只读锁定命令，防止导出数据库的时候有数据写入
				>unlock tables; #解除锁定

	* 导入数据库到MySQL从服务器
			

			$ mysql -u root -p #进入从服务器MySQL控制台
				>create database youngdb; #创建数据库
				>use youngdb #进入数据库
			$ source /home/youngdbbak.sql #导入备份文件到数据库
			$ mysql -u youngbak -h 192.168.137.226 -p #测试在从服务器上登录到主服务器

3. 配置MySQL主服务器的配置文件(my.cnf)


		$ vim /etc/my.cnf #在[mysqld]部分添加下面内容
			server-id = 1 # 设置服务器id，1表示为主服务器，如配置文件原有就不用添加
			log-bin = mysql-bin #启动MySQL二进制日志文件，配置文件中若有则不用添加
			binlog-do-db=youngdb #设置需要同步的数据库名称，如有多个数据库，可重复此参数，每个库一行
			binlog-ignore-db=mysql #设置不同步的MySQL系统数据库

		
	* 保存退出后，执行`$ service mysqld restart`重启MySQL
	* 进入MySQL控制台：


			$ mysql -u root -p
				>show variables like 'server_id'; #查看server-id是否为1
				> 
					+---------------+-------+
					| Variable_name | Value |
					+---------------+-------+
					| server_id | 1 |
					+---------------+-------+
					1 row in set (0.00 sec)
				>show master status;

					+------------------+----------+--------------+------------------+

					| File | Position | Binlog_Do_DB | Binlog_Ignore_DB |

					+------------------+----------+--------------+------------------+

					| mysql-bin.000011 | 107 | youngdb | mysql |

					+------------------+----------+--------------+------------------+

					1 row in set (0.00 sec) #注意，在这里需要记住File名(mysql-bin.000011)和Position(107)，不同的库位置不一定一样。在后面设置从库的时候需要用到

4. 配置MySLQ从服务器(192.168.137.179)

	* 编辑配置文件，在[mysqld]部分添加以下内容：


			server-id=2 #设置服务器id，修改为除了1以外的任何数字，表明这个MySQL是从库
			log-bin=mysql-bin
			replicate-do-db=youngdb #设置需要同步的数据库名
			replicate-ignore-db=mysql #不同步MySQL系统数据库
			read_only #设置只读

	* 完成后重启MySLQ服务，查看相关参数


			$ service mysqld restart #重启
			$ mysql -u root -p
				>show variables like 'server_id';
				>+---------------+-------+

				 | Variable_name | Value |

				 +---------------+-------+

				 | server_id | 2 |

				 +---------------+-------+

				 1 row in set (0.01 sec)

				>slave stop; #停止slave同步进程
				>change master to master_host='192.168.137.226',master_user='youngdbbak',master_password='123456',master_log_file='mysql-bin.000011' ,master_log_pos=107; #执行同步语句master_log_file和master_log_pos均为刚才在主服务器上记录的文件以及位置
				>slave start; #开启slave同步进程
				>SHOW SLAVE STATUS\G #查看slave同步信息，出现以下内容
				>*************************** 1. row ***************************

					Slave_IO_State: Waiting for master to send event

					Master_Host: 192.168.137.226

					Master_User: youngdbbak

					Master_Port: 3306

					Connect_Retry: 60

					Master_Log_File: mysql-bin.000011

					Read_Master_Log_Pos: 107

					Relay_Log_File: mysqlslave-relay-bin.000004

					Relay_Log_Pos: 253

					Relay_Master_Log_File: mysql-bin.000011

					Slave_IO_Running: Yes # 注意查看这一行以及下一行，均为yes表明主从设置成功

					Slave_SQL_Running: Yes
					.
					.
					.
					.
					mysql>

### 四、测试 ###

1. 进入MySQL主服务器(192.168.137.226)


		$ mysql -u root -p
			>use youngdb; #启用数据库
			>CREATE TABLE test(id int not null primary key,name char(20)); #创建test表

2. 进入从服务器


		$ mysql -u root -p
			>use youngdb;
			>show tables; #查看youngdb表结构，会看到新建的表，表明同步成功
			>+----------------------+

			 | Tables_in_youngdb |

			 +----------------------+

			 | test |

			 +----------------------+

			 1 row in set (0.00 sec)

### 五、总结 ###

MySQL的主从同步功能非常强大，配置也很灵活，配置的时候注意需要指明同步的以及忽略的数据库，设置好server-id避免冲突。在这个功能的基础上，搭配其他的监测工具可以实现更加完善的高可用MySQL架构。比如MySQL+MHA，MySLQ+LVS+Keepalived等方案。
