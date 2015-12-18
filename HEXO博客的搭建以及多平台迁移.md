title: HEXO博客的搭建以及多平台迁移
date: 2015-12-16 10:31:08
tags: [HEXO,Git]
---


> **前言**：
>最近这段时间发生的事情很多，也觉得自己越来越浮躁，出于练习以及实践，准备对最近自己所做的实践进行总结，也就有了这个博客，国际惯例，第一篇正式博客就从搭建这个博客说起吧。


## 一、 准备 ##
### Git的安装 ###
#### 1、在自己的github上添加仓库(repository) ####

* 项目命名规则为你的github的ID.github.io

#### 2、在自己的电脑上安装两个软件git和nodejs ####

建议先安装git，然后在git提供的shell下安装nodejs（win下和ubuntu下类似）


- **git的安装**
 + windows：下载并安装[git](http://git-scm.com/)或者配置官网的github客户端
 + ubuntu：执行`$ sudo apt-get install git-core`
- **nodejs的安装**
 + windows:下载[安装程序](https://nodejs.org/en/)并安装
 + Ubuntu下：` $ wget -qO- https://raw.github.com/creationix/nvm/master/install.sh | sh `或者下载源码包编译安装


<!--more-->


#### 3、生成SSH keys ####

在上述搭建完的环境下，执行以下命令生成密钥：

` $ ssh-keygen -t rsa -C "此处填写你的github账户" `

执行完了之后，会在你的根目录下生成两个文件，名称分别是id_rsa.和id_rsa.pub。得到这里两个文件之后，就在你的github网站上找到sittings->SSH keys->Add SSH key。然后将id_rsa.pub里面的内容用notepad++打开拷贝下来(最好用非windows的编辑器)，粘贴到Key里面，然后名字随便起就ok了

![class names](/img/SSH.png)

添加密钥成功后，就在本机上测试，在git提供的shell下执行` ssh -T git@github.com `当终端返回：

		 Hi "your account" !You've successfully authenticated, but GitHub does not provide shell access.

就表明连接成功了。

#### 4、安装Hexo ####

以上三步完成后，就可以使用npm安装Hexo了。

` $ npm install -g hexo-cli`

#### 5、建站 ####

Hexo安装完成后，执行下列命令，Hexo就会在指定文件夹生成需要的文件。


		 $ hexo init <folder> #folder若不指定，则在当前目录下生成
		 $ cd <folder>
		 $ npm install

完成上述步骤后，在目录下就会生成以下文件

		.
		├── _config.yml
		├── package.json
		├── scaffolds
		├── source
		|   ├── _drafts
		|   └── _posts
		└── themes


#### 6、文章撰写 ####

在上述命令执行之后，就可以开始写文章了。在shell下执行` hexo new 文章名称 `(这个是最简洁的，还有其他参数可以参考[Hexo帮助文档](https://hexo.io/zh-cn/docs/writing.html)。执行完了此命令后，就可以在source/_posts下看到有个后缀为.md的文档生成。打开此文档，就可以开始愉快的博文写作了，Hexo支持markdown语法。所以编写文档很便捷，可以参考[这里](http://www.jianshu.com/p/21d355525bdf)熟悉。

#### 7、文章的预览 ####

编写并且保存完博客之后，执行

		$ hexo generate
		$ hexo server
就可以在本地预览自己编写的博客，地址是localhost:4000


#### 7、配置 ####

到上述生成的目录下，找到_config.yml文件，用非windows的记事本打开，如notpad++，找到最后一部分，如下填写（每个冒号后面都有个空格，不然会报错）：

		deploy: git
		repo: https://github.com/youraccountname/youraccountname.git.io
		branch: master

然后保存。就可以开始文档的编辑了，这个步骤的目的是将生成的静态页面的相关文档上交到你的github账户下的仓库里。

#### 8、部署 ####

完成以上步骤之后，基本就完成博客的编写，但是仅仅是将文件生成好，别人还是没办法看到你所写的内容。这时候就要页面相关的文件部署到github仓库中。需要执行下面的语句：

		$ hexo deploy
至此就完成了Hexo博客的搭建了。

#### 9、迁移 ####

当我需要转移写作环境的时候怎么办?这个问题我想了很久额，在高人指点下。才想到这么个办法，这个办法仅供参考，如果你有更好的方法欢迎告诉我改进。我想到的方法是：

* 再创建一个仓库
* 每次完成了一篇博文就用git上传到该仓库里
* 当迁移到新的环境下，比如我到ubuntu系统下，就可以将该仓库的文件clone下来，然后再执行

		$ hexo init
		$ hexo generate
		$ hexo deploy
> 后记：第一篇博客花的时间有点儿长，都是因为自己不熟悉markdown。以后还得多写写，把这个养成一种习惯。能看到这儿的人，道个谢，共勉哈！



