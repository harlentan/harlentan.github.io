---
layout: post
title: JavaScriptCore编程之编译JavaScriptCore
tags: [blog]
keywords: JavaScriptCore编程之编译JavaScriptCore, JavaScriptCore编程, 编译JavaScriptCore, Ubuntu 编译JavaScriptCore
description: JavaScriptCore编程之编译JavaScriptCore 详细讲述了JavaScriptCore在Ubuntu下编译步骤
comments: true
---
JavaScriptCore在Ubuntu 12.04下的编译步骤。JavaScriptCore的历史原因，使得JavaScriptCore单独编译比较麻烦。本文讲解了如何单独编译JavaScriptCore

<!--more-->

##JavaScriptCore源码下载
JavaScriptCore源码下载地址:   
[https://github.com/harlentan/JavaScriptCore](https://github.com/harlentan/JavaScriptCore)
也可以前往WebKit的Apple官网下载nightly build的代码[http://nightly.webkit.org/](http://nightly.webkit.org/) 
##Ubuntu 12.04
##依赖
    * glib 2.3.8 or lader
    * libsoup 2.4
    * gtk+-3.0
    * cmake
    * libgstreamer
##编译步骤
之前几乎所有的开发者在Ubuntu上编译WebKit都会选择编译QtWebKit,编译完之后会产生libjavascriptcore.so. 但是随着Qt将WebKit替换成blink, webkit.org也删除了QtWebKit的编译步骤。
所以，这里我们只能选择gtk版本的。

具体编译脚本在[https://github.com/harlentan/JavaScriptCore](https://github.com/harlentan/JavaScriptCore)
###执行脚本
    cd JavaScriptCore
    ./build-jsc  --gtk

如果是在webkit.org上下载的代码：
则需要在在WebKit根目录下执行：
    .Tools/scripts/build-jsc
需要注意的是，里面需要一些版本比较新的库。所以最好是选择下载源码然后编译安装。

##Android
coming soon.