---
layout: post
title: 	Xapian Omega搭建自己的搜索引擎
tags: [搜索引擎技术]
keywords: Xpian, Omega, Xapian-omega, Sphinx, Lucene
description: 一直以来，对搜索引擎技术非常感兴趣，也希望做一些搜索引擎相关的技术。在过去的一年中，业余时间对于搜索引擎技术进行了学习。打算以Xapian为切入点，对搜索引擎技术进行学习和研究。在一开始较长的一段时间里，我都在努力尝试着将Xapian一直到的Android，后来因为效率资源等各种原因放弃了这个方案。
category: 搜索引擎
comments: true
share: true
---

<center>![Alt Text](/images/xapian-logo.png)</center>


一直以来，对搜索引擎技术非常感兴趣，也希望做一些搜索引擎相关的技术。在过去的一年中，业余时间对于搜索引擎技术进行了学习。打算以Xapian为切入点，对搜索引擎技术进行学习和研究。在一开始较长的一段时间里，我都在努力尝试着将Xapian一移植到Android，后来因为效率资源等各种原因放弃了这个方案。

##为什么选择Xapian?

在搜索引擎界，Lucene算得上是老牌的全文检索引擎了，Lucene是用Java写的，由于我个人对于Java不是很熟，所以没有选择Lucene。C++的开源全文检索引擎有Xapian和Sphinx。确切的说，我是先知道有Xapian的，后来才知道有Sphinx的。网上也有很多对于Xapian和Sphinx的比较，Xapian由于维护人员少以及参考资料少而颇为人诟病。跟Sphinx相比，Xapian一方面是实时性支持的不错，而且是嵌入式的，另一方面是内存占用更少。我选择Xapian，并不是因为产品驱动的，而仅仅是因为我个人业余喜欢倒腾新领域，最终选择Xapian，主要是跟其他检索引擎相比，代码量不是很大，另一方面，Xapian自带的omega可以迅速的搭建起一个搜索引擎，对于研究和学习来说，很方便。

##Xapian Omega搭建自己的搜索引擎

Xapian可以到这里[Xapian](https://github.com/xapian/xapian)进行下载：

	git clone https://github.com/xapian/xapian.git

对Xapian整个工程进行编译：
{% highlight bash  %}
cd Xapian
./bootstrap
./configure
make
sudo make install
{% endhighlight %}
Omega的代码位于Xapian/xapian-applications中，上面的编译也会对Omega进行编译。   

另外，Xapian-Omega搜索引擎需要Apache或者httpd的支持。我的系统是Ubuntu 12.04，我是用的是Apache2。   

在Xapian-Omega都编译安装完毕之后，接下来需要，需要对索引环境进行配置：
{% highlight bash  %}
cp /usr/local/lib/xapian-omega/bin/omega /usr/lib/cgi-bin/omega
cd Xapian/xapian-applications/
cp omega.conf /usr/lib/cgi-bin/
chmod 755 /usr/lib/cgi-bin/omega.cgi
{% endhighlight %}
需要注意的是，有些资料上说直接复制Xapian/xapian-applications/omega/omega到/usr/lib/cgi-bin/下面。经过我的多次实践，Xapian/xapian-applications/omega/omega实际上是无法使用的，会导致后面的cgi无法正常工作。  


刚刚cp的omega.conf的内容如下：

	database_dir /var/lib/omega/data
	template_dir /var/lib/omega/templates
	log_dir /var/log/omega
	cdb_dir /var/lib/omega/cdb
这些目录，在使用omega之前，需要我们先手动创建起来：

	mkdir -p /var/lib/omega/data
	mkdir /var/lib/omega/templates
	mkdir /var/lib/omega/cdb
	mkdir /var/log/omega

复制template：

	cd Xapian
	cp templates/* /var/lib/omega/templates

到这里，所有的索引环境都已经搭建完毕，接下来需要对文件进行索引了，可以自己准备一些文件，也可以下载这个数据包进行索引[http://fayettedigital.com/book/book.0.1.tar.gz](http://fayettedigital.com/book/book.0.1.tar.gz)，这里，我使用这个数据包进行演示，将这个数据压缩包解压到 /var/www/book下面，然后对数据进行索引：

	/usr/local/bin/omindex --db /var/lib/omega/data/default --url /book /var/www/book
其中 --db /var/lib/omega/data/default是用来创建一个名叫default的数据库，--url是用来指明需要索引的内容的路径。   

在浏览器中输入http://localhost/cgi-bin/omega，进入搜索页面之后搜索一些关键字，搜索结果如下：

<center>![Alt Text](/images/xapian-omega-demo.png)</center>
