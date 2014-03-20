---
layout: post
title: JavaScriptCore编程详解--JavaScriptCore初探
tags: [blog]
keywords: JavaScriptCore编程详解, JavaScriptCore编程, 深入浅出JavaScriptCore编程
description: JavaScriptCore编程详解系列主要针对自己工作中对JavaScriptCore的理解，详细讲解了JavaScriptCore编程以及以及JavaScriptCore内部原理，深入浅出JavaScriptCore编程，由浅到深的方式讲解JavaScriptCore编程。
category: C/C++, JavaScriptCore
comments: true
---

JavaScriptCore编程详解系列主要针对自己工作中对JavaScriptCore的理解，详细讲解了JavaScriptCore编程以及JavaScriptCore内部原理，深入浅出JavaScriptCore编程，由浅到深的方式讲解JavaScriptCore编程。

<!--more-->

##前言
2012写过一系列博客《Google V8编程详解》,发表在CSDN上，效果非常不错。起初本来预计跟出版社出一本《Google V8编程详解》,后由于后来项目很紧，几乎没时间继续把书写下去，所以没能弄完。现在博客从CSDN搬家到在这里，一开始打算《Google V8编程详解》系列也迁过来，最终决定就放CSDN好了。如果大家有兴趣，可以前往我的[CSDN博客专栏](http://blog.csdn.net/feiyinzilgd)。从2013年下半年开始，工作从V8切换到了JavaScriptCore。说实话，JavaScriptCore的资料确实太少了，而且Apple的工程师写的JavaScriptCore代码注释很少，很多用法以及特性都需要自己摸索，看具体的实现才能明白。今天刚刚把《JavaScriptCore编程详解》的计划安排出来，JavaScriptCore编程详解系列文章将结合JavaScriptCore的实现来讲解。需要指出的是，Apple官方有一份[JavaScriptCore Framework Reference](JavaScriptCore Framework Reference), 这是JavaScriptCor的C接口的API,C的API其实是JavaScriptCore C++ API的封装。我主要是将C++ API的部分。C++ API明白了，自然也就理解C API了。我会在一些关键技术点上，稍微结合C的API进行讲述。

##
