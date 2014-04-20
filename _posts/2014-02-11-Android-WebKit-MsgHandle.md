---
layout: post
title: Android WebKit消息处理
tags: [WebKit]
keywords: WebKit, Android WebKit, WebView, WebKit 消息处理, WebKit消息分发, Android WebKit消息处理
description: Android WebKit消息处理详细讲述了Android WebKit中WebView, WebViewClassic, WebViewCore, WebViewInputDispatcher中消息传递与处理
alias: /2014/02/11/Android-WebKit-MsgHandle.html‎
comments: true
---

在Android API中，用WebView的形式向开发者提供WebKit的接口以及特性。WebView其实是对WebKit的封装以及扩展，在android4.4里面，已经WebKit已经换成Chromium(在后续博客中会对android4.4
的WebView/Chromium进行详细讲解)。

<!--more-->

### WebKit消息处理框架的搭建
整个WebKit主要分为2个线程，一个是Ui线程，也就是应用程序使用WebView所在的主线程，另一个WebCore线程。WebView的消息处理，主要是Ui线程和WebCore线程的交互。一部分Ui线程向WebCore发送的命令操作消息，例如LOAD_URL，另一部分是来自Ui的touch消息。

主要涉及到的类如图所示：
![Alt text](/images/webkitjava.jpg)
其中，WebViewClassic是WebView的Provider，或者说是delegate，凡是涉及到消息处理或者需要跟WebCore交互的接口，都是直接调用WebViewClassic的同名函数。
WebViewInputDispatcher就是用来处理Ui的touch事件的。后面会专门讲解WebKit的touch事件传递以及处理流程的。本章就不展开讲解了。

### WebView的初始化

时序图如下：
![Alt text](/images/webkiteventsequens.png)
上图其实就是WebView/WebKit的初始化流程，主要是对WebViewClassic, WebCore, WebViewInputDispatcher进行初始化，为WebKit资源请求前做准备。WebView构造函数中会调用createWebView，这个过程其实就是创建WebView的provider，实际返回的对象就是WebViewClassic，紧接着init初始化WebViewClassic，在WebViewClassic的init中会new WebViewCore()，之后对WebViewCore进行初始化，在WebViewCore自己初始化完毕之后，表明现在WebCore已经可以处理来自WebView的消息了，包括touch事件，此时WebViewCore会发送消息给WebViewClassic，也就是上图中看到的WEBCORE\_INITIALIZED\_MSG_ID，WebViewClassic收到此消息之后，就会初始化touch事件的分发器：WebViewInputDispatcher。这个流程结束之后，应用就可以通过WebView，loadUrl了。

从这个图中，我们看到，在WebCore的构造函数中会new WebCoreThread，这个线程就是上面提到的WebCore线程。到此为止，我们已经可以看到有2个线程了，一个是WebView所在的Ui线程，另一个就是WebCore线程。

那么WebKit的消息处理究竟怎么工作的呢？
在WebView中，消息在线程间的分发使用的是Handler。在WebKit的消息分发机制中的总共有三个Handler如下：
![Alt text](/images/webkithandlers.jpg)
#### mPrivateHandler：
在WebViewClassic中创建，用来分发和处理UI相关消息，例如重绘，touch事件，另外就是负责跟WebCore线程交互。
#### sWebCoreHandler：
在WebCoreThread中间，主要负责WebViewCore初始化、WebViewCoreWatchDog心跳、WebCore线程优先级别调整。
#### mHandler：
WebCore线程消息循环最主要的handler。任何需要调用WebCore接口的，都需要通过mHandler  send到WebCore线程中去。
在EventHub的transferMessages()中被new出来的，由于transferMessages()实在WebViweCore的initialize()中被调用的，所以，EventHub的mHandler也是在WebCore线程中。

Ui线程第一次向WebCore线程发送的消息，并没有直接被分发到WebCore线程中去。而是被缓存在WebViewCore中的mMessages list中，因为有可能在WebKit的消息处理框架还未初始化完毕，Ui线程就已经开始向WebCore线程发送消息了。所以，当WebViewCore最后初始化完毕之后，会调用transferMessages()，在transferMessages中将mMessages中的消息通过mHandler全部send到WebCore线程中去。

到这里已经应该很明白了：
Ui线程的消息通过WebViewClassic的mPrivateHandler处理。WebCore线程的消息通过EventHub的mHandler和WebViewCore的sWebCoreHandler处理。各个Handler之间可以相互send消息到对方的消息队列中去。