title: Nginx性能调优初识
date: 2016-01-14 17:01:57
tags: [Nginx,性能调优]
---

##  一、简介  ##

Nginx的轻量级、高可用以及高性能等诸多优点，让它成为诸多架站web服务器的选择热门之一。在前面的博文介绍了LNMP网站架构的搭建之后，在此初步描述下Nginx的配置文件以及初步的调优方法。

<!--more-->


## 二、配置文件 ##

在Nginx的配置文件里大致的结构如下：


	vi /usr/local/nginx/conf/nginx.conf		

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

## 三、配置文件参数详解以及优化设置 ##

调优的参数大致如下：

1. worker_processes
2. worker_connections
3. worker_rlimit_nofile
4. Buffer
5. Timeouts
6. Ggzip Compression
7. Static File Caching
8. logging

### 1. worker_processes ###


`worker_process`表述Nginx的工作进程数目，一般设置为CUP核心的数目即可。查看CUP核心的数目指令可用这个command：`grep processor /proc/cpuinfo | wc -l`


### 2. worker_connections ###


`worker_connections`表示每个nginx进程允许的最大用户连接数(并发连接数)，默认值为1024.理论上每台Nginx服务器最大连接数为N=worker_processes*worker_connections


### 3. worker rlimit nofile ###


`worker_rlimit_nofile`指的是每个nginx进程所能打开的最多文件描述符数，理论值是`ulimit -n`除以`worker_processes`


### 4.Buffer ###

Buffer是一个很重要的参数，如果设置过小，Nginx就会不断地写临时文件，进而造成磁盘不停地读写，增大磁盘IO.
`client_body_buffer_size`指的是客户端请求的最大单个文件字节数
`client_header_buffer_size`设置客户端请求的Header头缓冲区大小，一般为1K.
`client_max_body_size`设置客户端能够上传的文件大小，默认为1M.
`large_client_header_buffers`该参数用于设置客户端请求的Header头缓冲区大小.
具体参考配置如下：

		
	client_body_buffer_size		10k;
	client_header_buffer_size	1k;
	client_max_body_size		8m;
	largr_client_header_buffers	2 1k;

### 5. Timeouts ###

`client_body_timeout`和`client_body_timeout`分别设置请求头和请求体的超时时间，如果没有发送请求头和请求体，Nginx服务器会返回408错误或者request time out.
`keepalive_timeout`给客户端分配keep-alive连接超时时间.
`send_timeout`为客户端响应时间，这个设置不会用于整个转发器，而是在两侧客户端读取操作之间，若这段时间内，客户端没有读取数据，Nginx就会关闭连接.
参考配置如下：

	
	client_body_timeout		12;
	client_header_timeout	12;
	keepalive_timeout		15;
	send_timeout			10;

### 6. Gzip Compression ###

开启Gzip，可以帮助Nginx减少网络传输工作，此外还需要注意`gzip_comp_level`的设置，不宜过高，否则Nginx会浪费CPU的执行周期.
参考配置如下:

	gzip				on;
	gzip_comp_level		2;
	gzip_min_length		1000;
	gzip_proxied		expired no-cache no-store private auth;
	gzip_types			tex/plain application/x-javascript text/xml text/css application/xml;

### 7. Static File Caching ###


参考配置：

	location ~* .(jpg|jpeg|png|gif|ico|css|js)$ {
		expires 365d;
	}

以上文件类型可以根据Nginx服务器匹配增加或者减少。

### 8. logging ###

`access_log`设置Nginx是否开启存储访问日志，关闭此选项，可以提升磁盘IO。对应配置部分如下：

	access_log		off;



至此介绍完Nginx的性能调优的小部分内容。在以后如果有更多的方法，都会在此文章中更新。