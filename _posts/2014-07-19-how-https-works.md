---
layout: post
title: Https(SSL/TLS)原理详解
tags: [Network, C/C++]
keywords: Https, SSL, TLS, Master secret, Master key, PreMaster Key, PreMaster secret, Https原理握手, SSL/TLS原理握手
description: Https，是一种基于SSL/TLS的Http，所有的http数据都是在SSL/TLS协议封装之上传输的。Https协议在Http协议的基础上，添加了SSL/TLS握手以及数据加密传输，也属于应用层协议。所以，研究Https协议原理，最终其实是研究SSL/TLS协议。其中的PreMaster Key(PreMaster secret), Master key(Master secret)，Session key是整个环节中比较重要的部分。
category: Network
comments: true
share: true
---

最近开始做Https网络方面的工作，花时间学习了下 Https，SSL/TLS相关的内容。把我对于Https，SSL/TLS的理解跟大家分享下，顺便埋个伏笔，时机成熟之后还要跟大家分享下[《加解密基础知识》]()  ，因为SSL/TLS有很多加解密方面的知识。在技术方面，我对于我自己的要求是所到之处，必须深入理解。对于理解不透彻或者有误的地方，欢迎大家参与讨论。

##概述

Https(Hyper Text Transfer Protocol over Secure Socket Layer)，是一种基于SSL/TLS的Http，所有的http数据都是在SSL/TLS协议封装之上传输的。Https协议在Http协议的基础上，添加了SSL/TLS握手以及数据加密传输，也属于应用层协议。所以，研究Https协议原理，最终其实是研究SSL/TLS协议。

SSL协议，是一种安全传输协议，最初是由 Netscape 在1996年发布，由于一些安全的原因SSL v1.0和SSL v2.0都没有公开，直到1996年的SSL v3.0。TLS是SSL v3.0的升级版，目前市面上所有的Https都是用的是TLS，而不是SSL。本文主要分析和讲解TLS。


##TLS握手

TLS的握手阶段是发生在TCP握手之后。握手实际上是一种协商的过程，对协议所必需的一些参数进行协商。TLS握手过程分为四步，过程如下：（备注：图中加方括号的均为可选消息）
<center>![Alt Text](/images/TLS.svg)</center>

###Client Hello

由于客户端(如浏览器)对一些加解密算法的支持程度不一样，但是在TLS协议传输过程中必须使用同一套加解密算法才能保证数据能够正常的加解密。在TLS握手阶段，客户端首先要告知服务端，自己支持哪些加密算法，所以客户端需要将本地支持的加密套件(Cipher Suite)的列表传送给服务端。除此之外，客户端还要产生一个随机数，这个随机数一方面需要在客户端保存，另一方面需要传送给服务端，客户端的随机数需要跟服务端产生的随机数结合起来产生后面要讲到的Master Secret。

###Server Hello

上图中，从Server Hello到Server Done，有些服务端的实现是每条单独发送，有服务端实现是合并到一起发送。Sever Hello和Server Done都是只有头没有内容的数据。

服务端在接收到客户端的Client Hello之后，服务端需要将自己的证书发送给客户端。这个证书是对于服务端的一种认证。例如，客户端收到了一个来自于称自己是www.alipay.com的数据，但是如何证明对方是合法的alipay支付宝呢？这就是证书的作用，支付宝的证书可以证明它是alipay，而不是财付通。证书是需要申请，并由专门的数字证书认证机构(CA)通过非常严格的审核之后颁发的电子证书。颁发证书的同时会产生一个私钥和公钥。私钥由服务端自己保存，不可泄漏。公钥则是附带在证书的信息中，可以公开的。证书本身也附带一个证书电子签名，这个签名用来验证证书的完整性和真实性，可以防止证书被串改。另外，证书还有个有效期。

在服务端向客户端发送的证书中没有提供足够的信息的时候，还可以向客户端发送一个Server Key Exchange。   

此外，对于非常重要的保密数据，服务端还需要对客户端进行验证，以保证数据传送给了安全的合法的客户端。服务端可以向客户端发出Cerficate Request消息，要求客户端发送证书对客户端的合法性进行验证。

跟客户端一样，服务端也需要产生一个随机数发送给客户端。客户端和服务端都需要使用这两个随机数来产生Master Secret。

最后服务端会发送一个Server Hello Done消息给客户端，表示Server Hello消息结束了。


###Client Key Exchange

如果服务端需要对客户端进行验证，在客户端收到服务端的Server Hello消息之后，首先需要向服务端发送客户端的证书，让服务端来验证客户端的合法性。

在此之前的所有TLS握手信息都是明文传送的。在收到服务端的证书等信息之后，客户端会使用一些加密算法(例如：RSA, Diffie-Hellman)产生一个48个字节的Key，这个Key叫PreMaster Secret，很多材料上也被称作PreMaster Key, 最终通过Master secret生成session secret， session secret就是用来对应用数据进行加解密的。PreMaster secret属于一个保密的Key，只要截获PreMaster secret，就可以通过之前明文传送的随机数，最终计算出session secret，所以PreMaster secret使用RSA非对称加密的方式，使用服务端传过来的公钥进行加密，然后传给服务端。

接着，客户端需要对服务端的证书进行检查，检查证书的完整性以及证书跟服务端域名是否吻合。

ChangeCipherSpec是一个独立的协议，体现在数据包中就是一个字节的数据，用于告知服务端，客户端已经切换到之前协商好的加密套件的状态，准备使用之前协商好的加密套件加密数据并传输了。

在ChangecipherSpec传输完毕之后，客户端会使用之前协商好的加密套件和session secret加密一段Finish的数据传送给服务端，此数据是为了在正式传输应用数据之前对刚刚握手建立起来的加解密通道进行验证。


###Server Finish

服务端在接收到客户端传过来的PreMaster加密数据之后，使用私钥对这段加密数据进行解密，并对数据进行验证，也会使用跟客户端同样的方式生成session secret，一切准备好之后，会给客户端发送一个ChangeCipherSpec，告知客户端已经切换到协商过的加密套件状态，准备使用加密套件和session secret加密数据了。之后，服务端也会使用session secret加密后一段Finish消息发送给客户端，以验证之前通过握手建立起来的加解密通道是否成功。  


根据之前的握手信息，如果客户端和服务端都能对Finish信息进行正常加解密且消息正确的被验证，则说明握手通道已经建立成功，接下来，双方可以使用上面产生的session secret对数据进行加密传输了。


##Secret Keys

上面的分析和讲解主要是为了突出握手的过程，所以PreMaster secret，Master secret，session secret都是一代而过，但是对于Https，SSL/TLS深入的理解和掌握，这些Secret Keys是非常重要的部分。所以，准备把这些Secret Keys抽出来单独分析和讲解。

我们先来看看这些Secret Keys的的生成过程以及作用流程图：

<center>![Alt Text](/images/tls-keys-create.svg)</center>



##PreMaster secret

PreMaster secret是在客户端使用RSA或者Diffie-Hellman等加密算法生成的。它将用来跟服务端和客户端在Hello阶段产生的随机数结合在一起生成Master secret。在客户端使用服务单的公钥对PreMaster secret进行加密之后传送给服务端，服务端将使用私钥进行解密得到PreMaster secret。也就是说服务端和客户端都有一份相同的PreMaster secret和随机数。

PreMaster secret前两个字节是TLS的版本号，这是一个比较重要的用来核对握手数据的版本号，因为在Client Hello阶段，客户端会发送一份加密套件列表和当前支持的SSL/TLS的版本号给服务端，而且是使用明文传送的，如果握手的数据包被破解之后，攻击者很有可能串改数据包，选择一个安全性较低的加密套件和版本给服务端，从而对数据进行破解。所以，服务端需要对密文中解密出来对的PreMaster版本号跟之前Client Hello阶段的版本号进行对比，如果版本号变低，则说明被串改，则立即停止发送任何消息。

关于PreMaster Secret(Key)的计算请参考[《Htttps SSL/TLS PreMaster/Master Secret(Key)计算》](http://www.fenesky.com/blog/2014/07/25/how-premaster-secret.html)。


##Master secret

上面已经提到，由于服务端和客户端都有一份相同的PreMaster secret和随机数，这个随机数将作为后面产生Master secret的种子，结合PreMaster secret，客户端和服务端将计算出同样的Master secret。   

Master secret是有系列的hash值组成的，它将作为数据加解密相关的secret的Key Material。Master secret最终解析出来的数据如下：

<center>![Alt Text](/images/tls-keys.svg)</center>

其中，write MAC key，就是session secret或者说是session key。Client write MAC key是客户端发数据的session secret，Server write MAC secret是服务端发送数据的session key。MAC(Message Authentication Code)，是一个数字签名，用来验证数据的完整性，可以检测到数据是否被串改。关于MAC的工作原理详见[MAC](http://en.wikipedia.org/wiki/Message_authentication_code)。

关于Session Secret(Key)的计算请参考[《Htttps SSL/TLS Session Secret(Key)计算》](http://www.fenesky.com/blog/2014/07/25/how-session-secret.html)。


##应用数据传输

在所有的握手阶段都完成之后，就可以开始传送应用数据了。应用数据在传输之前，首先要附加上MAC secret，然后再对这个数据包使用write encryption key进行加密。在服务端收到密文之后，使用Client write encryption key进行解密，客户端收到服务端的数据之后使用Server write encryption key进行解密，然后使用各自的write MAC key对数据的完整性包括是否被串改进行验证。


##总结

讲到这里，Https的原理，实际上是SSL/TLS的原理都讲解完毕，我只能说TLS不仅是一个安全传输协议，而且是一个艺术品。