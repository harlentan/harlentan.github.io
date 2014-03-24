---
layout: post
title: JavaScriptCore编程详解 之 JavaScriptCore初探
tags: [blog]
keywords: JavaScriptCore编程详解, JavaScriptCore编程, 深入浅出JavaScriptCore编程
description: JavaScriptCore编程详解系列主要针对自己工作中对JavaScriptCore的理解，详细讲解了JavaScriptCore编程以及以及JavaScriptCore内部原理，深入浅出JavaScriptCore编程，由浅到深的方式讲解JavaScriptCore编程。
category: C/C++, JavaScriptCore
comments: true
---

JavaScriptCore编程详解系列主要针对自己工作中对JavaScriptCore的理解，详细讲解了JavaScriptCore编程以及JavaScriptCore内部原理，深入浅出JavaScriptCore编程，由浅到深的方式讲解JavaScriptCore编程。

<!--more-->

##前言
2012写过一系列博客《Google V8编程详解》,发表在CSDN上，效果非常不错。起初本来预计跟出版社出一本《Google V8编程详解》,后由于后来项目很紧，几乎没时间继续把书写下去，所以没能弄完。现在博客从CSDN搬家到在这里，一开始打算《Google V8编程详解》系列也迁过来，最终决定就放CSDN好了。如果大家有兴趣，可以前往我的[CSDN博客专栏](http://blog.csdn.net/feiyinzilgd)。从2013年下半年开始，工作从V8切换到了JavaScriptCore。说实话，JavaScriptCore的资料确实太少了，而且Apple的工程师写的JavaScriptCore代码注释很少，很多用法以及特性都需要自己摸索，看具体的实现才能明白。今天刚刚把《JavaScriptCore编程详解》的计划安排出来，JavaScriptCore编程详解系列文章将结合JavaScriptCore的实现来讲解。需要指出的是，Apple官方有一份[JavaScriptCore Framework Reference](https://developer.apple.com/library/mac/documentation/Carbon/Reference/WebKit_JavaScriptCore_Ref/_index.html), 这是JavaScriptCor的C接口的API,C的API其实是JavaScriptCore C++ API的封装。我主要是将C++ API的部分。C++ API明白了，自然也就理解C API了。我会在一些关键技术点上，稍微结合C的API进行讲述。

##JavaScriptCore初探
###现状
JavaScriptCore作为一款优秀的JavaScript引擎，目前主要运用与WebKit当中，在Google V8问世之前，是世面上应用最广泛的JavaScript引擎，但是随着Google V8的崛起，Qt之类的WebKit porting也已经替换成V8, 目前JavaScriptCore主要阵营基本就只剩下Apple, 但是Apple目前的safari内置的JavaScript引擎也使用的不是JavaScriptCore。这就使得JavaScriptCore的地位比较尴尬。跟WebKit主要还是通过IDL的方式，向浏览器binding接口。目前很难看到除了WebKit意外，独立使用JavaScriptCore的产品。 而我目前开发的nodejsc，就是使用JavaScriptCore作为JS引擎，JavaScriptCore版本的nodejs.
###基本概念
在JavaScriptCore的世界分为js部分和C++部分，js的对象在C++中都是JSValue, 所有需要向JS导出的对象都必须需间接或直接继承于JSCell或者JSObject. 所有的JS对象都会有一个C++对象与之对应。也就意味着，一个JS对象在C++里面，至少会产生三个主要对象： Constructor, Object, Prototype. 而且JS里面对象，原型，属性跟C++中的行为一样。例如：
{% highlight java linenos %}
var a = {};
a.x = 10;
{% endhighlight %}
在C++里面，也会相应的在该JS对象对应的C++对象上添加一个想x属性。   
在JavaSCriptCor里面，经常会看到一个特殊的对象: ExecState* state. 这个对象往往都作为函数的第一个参数。ExecState的原始定义如下：
{% highlight C++ %}
typedef CallFrame* ExecState;
{% endhighlight %}
从其原型就可以看出，ExecState是一个叫CallFrame的东西，它其实是保存这当前JS运行状态的一个变量，包括参数等等。  
###内存管理
所有的JS对象，都是在JavaScriptCore的Heap上分配并管理的。 而且在JavaScriptCore的Heap上创建对象的时候，都需要使用replacement new的方式。在JavaScriptCore析购这些对象的时候，不会调用该对象的析购函数，而是调用对象的destroy函数。JavaScriptCore最终分配内存的方式是使用mmap。这样就降低了频繁分配内存的开销。JavaScriptCore对与内存申请做了很多的优化，使的创建对象的过程变得很轻量级，很多时候，当需要创建一个JS对象的时候，根本就不用重新分配内存，只需要简单的取一个空闲的对象，替换对象里面内容即可。关于JavaScriptCore内存管理方面，后面会有专门的章节详细的讲解。
