---
layout: post
title: Htttps SSL/TLS Session Secret(Key)计算
tags: [Network, C/C++]
keywords: Https, SSL, TLS, Master secret, Master key, PreMaster Key, PreMaster secret, Https原理握手, SSL/TLS原理握手
description: Https SSL/TLS的Session Key(Session Secret)计算流程演示。Session Secret(Key)是从Key Materaial中获取的。Key Material的计算跟Master Secret(Key)的计算类似，只不过计算的次数要多。Key Material需要计算12次，从而产生12个hash值，计算过程如图
category: Network
comments: true
share: true
---

Session Secret(Key)是从Key Materaial中获取的。Key Material的计算跟Master Secret(Key)的计算类似，只不过计算的次数要多。
Key Material需要计算12次，从而产生12个hash值，计算过程如图：


<center>![Alt Text](/images/session-secret.svg)</center>

由于画图展示的原因，上面我只画出了计算4个hash的流程，其实这个是要计算12次，来产生12个hash。

产生12个hash之后，紧接着就可以从这个Key Material中获取Session Secret了。

<center>![Alt Text](/images/key-material.svg)</center>

如图所示，从Key Material依次可以解析出Client MAC, Server MAC, Client Cipher，Server Cipher, Client IV, Server IV。
其中，根据前面讲解的[《Https(SSL/TLS)原理详解》](http://www.fenesky.com/blog/2014/07/19/how-https-works.html)， Client/Server MAC是用来对数据进行验证的，Cliet/Server Cipher就是Client/Server write encryption Key，用来对数据进行加解密的。