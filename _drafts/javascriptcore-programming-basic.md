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
2012写过一系列博客《Google V8编程详解》,发表在CSDN上，效果非常不错。起初本来预计跟出版社出一本《Google V8编程详解》,后由于后来项目很紧，几乎没时间继续写下去，所以没能弄完。现在博客从CSDN搬家到在这里，一开始打算《Google V8编程详解》系列也迁过来，最终决定就放CSDN好了。如果大家有兴趣，可以前往我的[CSDN博客专栏](http://blog.csdn.net/feiyinzilgd)。从2013年下半年开始，工作从V8切换到了JavaScriptCore。说实话，JavaScriptCore的资料确实太少了。很多东西都要靠自己探索。 而且Apple的工程师写的JavaScriptCore

##
