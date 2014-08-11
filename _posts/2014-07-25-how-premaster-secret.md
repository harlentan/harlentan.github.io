---
layout: post
title: Htttps SSL/TLS PreMaster/Master Secret(Key)计算
tags: [Network, C/C++]
keywords: Https, SSL, TLS, Master secret, Master key, PreMaster Key, PreMaster secret, Https原理握手, SSL/TLS原理握手
description: Https SSL/TLS的Master Key(PreMaster Secret)计算流程演示。很多人一开始研究Https SSL/TLS的时候，都很困惑PreMaster/Master Secret(Key)是如何被计算出来的，最近通过翻看其他资料以及openssl的源码，总结出PreMaster/Master Secret(Key)的计算流程如图所示
category: Network
comments: true
share: true
---

很多人一开始研究Https SSL/TLS的时候，都很困惑PreMaster/Master Secret(Key)是如何被计算出来的，最近通过翻看其他资料以及
openssl的源码，总结出PreMaster/Master Secret(Key)的计算流程如图所示：
<center>![Alt Text](/images/master-secret.svg)</center>

其中Client Random和Server Random都是在前面的[《Https(SSL/TLS)原理详解》](http://www.fenesky.com/blog/2014/07/19/how-https-works.html)中讲解过的，Client Hello 和Server Hello阶段都会发送各自的Random随机数给对方，最终都是用来计算Master Secret的。

至于PreMaster Secret(Key)的计算，主要是通过RSA或者Diffie-Hellman算法生成。我们可以看出，由于在Say Hello阶段，随机数都是明文传送的，如果PreMaster Secret泄漏的话，会导致整个SSL/TLS失效。
