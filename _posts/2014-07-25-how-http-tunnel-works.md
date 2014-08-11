---
layout: post
title: Http隧道(tunnel)技术与Proxy
tags: [Network, C/C++, Linux]
keywords: Http隧道, Http tunnel, Http Proxy, Https Proxy
description: 根据RFC2817的讲解发现，在使用Proxy请求https的时候，首先会使用HTTP的CONNECT Method向Proxy发起请求。另外，更具RFC2816中关于CONNECT Method的讲解，HTTP的CONNECT方法是用跟Proxy一起动态的建立HTTP隧道的。
category: Network
comments: true
share: true
---

一直都没有深入研究过 Http Proxy，最近在使用libcurl的过程中，发现在有Proxy的情况下，使用CURL请求一个https的资源，会有返回2个response。经过一番抓包和研究之后，发现另有原因。

根据[RFC2817](http://www.ietf.org/rfc/rfc2817.txt)的讲解发现，在使用Proxy请求https的时候，首先会使用HTTP的CONNECT Method向Proxy发起请求。

另外，更具[RFC2816](http://tools.ietf.org/html/rfc2616#section-9.9)中关于CONNECT Method的讲解，HTTP的CONNECT方法是用跟Proxy一起动态的建立HTTP隧道的。

<center>![Alt Text](/images/http-tunnel.svg)</center>

如图所示，通过Proxy，Client和Content Server之间建立起了隧道。这种隧道使得客户端感觉不到代理的存在，在客户端开来，它是直接跟要请求的资源服务器在通信。

Http隧道分为两种：

1. 不使用CONNECT的隧道
2. 使用CONNECT的隧道

不使用CONNECT的隧道，实现了数据包的重组和转发。也就是在我们使用Proxy的时候，然后发起Http请求，使用的就是非CONNECT的隧道。在这种情况下，在Proxy收到来自客户端的Http请求之后，会重新创建Request请求，并发送到目标服务器，也就是图中的Content Server。当目标服务器返回Response给Proxy之后，Proxy会对Response进行解析，然后重新组装Response，发送给客户端。所以，在不使用CONNECT方式建立的隧道，Proxy有机会对客户端与目标服务器之间的通信数据进行窥探，而且有机会对数据进行串改。

而对于使用CONNECT的隧道则不同。当客户端向Proxy发起Http CONNECT Method的时候，就是告诉Proxy，先在Proxy和目标服务器之间先建立起连接，在这个连接建立起来之后，目标服务器会返回一个回复给Proxy，Proxy将这个回复转发给客户端，这个Response是Proxy跟目标服务器连接建立的状态回复，而不是请求数据的Response。在此之后，客户端跟目标服务器的所有通信都将使用之前建立起来的建立。这种情况下的Http隧道，Proxy仅仅实现转发，而不会关心转发的数据。   

这也是为什么在使用Proxy的时候，Https请求必须首先使用Http CONNECT建立隧道。因为Https的数据都是经过加密的，Proxy是无法对Https的数据进行解密的，所以只能使用CONNECT，仅仅对通信数据进行转发。

所以，实际上所有的网络库(例如libcurl, chromium_net)在实现的时候，如果使用了Proxy，在请求Https协议的资源是，首先是使用CONNECT方法建立Http隧道，等收到目标服务器建立成功的回复之后，开始做SSL/TLS握手，然后进行数据传输。   

虽然Http CONNECT隧道安全，能够使得客户端的数据实现"翻墙", 但是Http CONNECT容易失控，由于数据都是加密的，Proxy跟目标服务器建立TCP连接之后，之后的所有数据都服用这个连接。但是Http CONNECT隧道不仅仅是可以建立Https 443端口的连接，实际上可以建立任何端口的连接，这也使得Proxy变得非常危险，所以，一般的服务器都会对Http CONNECT隧道进行控制，只能实现Https 443端口的隧道通信。


