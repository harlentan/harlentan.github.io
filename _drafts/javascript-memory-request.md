---
layout: post
title: JavaScriptCore对象创建与内存分配原理
tags: [blog]
keywords: JavaScriptCore对象创建与内存申请原理, JavaScriptCore内存, JavaScriptCore内存优化, JavaScriptCore, JavaScriptCore编程, JavaScriptCore内存分配, JavaScriptCore对象, JavaScriptCore Heap
description: JavaScriptCore对象创建与内存分配原理, 详细讲解了JavaScriptCore对象创建,内存分配以及内存管理的内部原理。对JavaScriptCore内存优化提供了深入的理论依据。
category: C/C++, JavaScriptCore
comments: true
---
主要针对JavaScriptCore的对象创建过程，讲解JavaScriptCore内存分配原理，包括：JavaScriptCore内存申请, JavaScriptCore内存管理有比较详细的分析，给JavaScriptCore内存优化提供深入的理论依据。  

<!--more-->

##前言
上一章讲解了[《Linux动态库的工作原理详解》](http://www.fenesky.com/blog/2014/03/17/how-shared-library-works.html)。本来打算继续讲解关于动态库的优化，但是由于本周主要做关于JavaScriptCore内存优化的工作，所以把自己对于JavaScriptCore内存方面的理解拿出来跟大家分享，也便于自己加深对这部分内容的理解和掌握。所以关于动态库的优化部分，将放在后面讲解。

##对象创建
