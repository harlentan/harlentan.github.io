---
layout: post
title: WebKit中JavaScriptCore对象模型
tags: [JavaScriptCore, WebKit]
keywords: WebKit, JavaScriptCore, JavaScript, WebCores
description: 一个进程中，可以有多个WebView，或者多个Frame，多个Document，但是一个进程中，只有一个VM。我前面的一些文章有提到过，每一个JS对象，都会在JavaScriptCore中，有一个C++ JSCell的子类对象与之对应。这些JSCell，都是分配在JavaScriptCore的Heap中。在JavaScriptCore中，一个VM对象，对应一个Heap对象。
category: JavaScriptCore
comments: true
share: true
---


从Chrome浏览器的插件说起。Chrome的插件是一种非常强大的浏览器扩展机制。插件中可以有JS文件，这些JS代码可以操作当前页面内的所有的HTML元素。举例来说，有道词典的Chrome插件，可以监听到鼠标选中的语句，然后进行翻译。稍微有点前端知识的，都知道，在所有的浏览器中，JS是无法夸Frame访问的，虽然我没有看过Chrome的插件实现机制，但是就凭这一点，可以猜测，Chrome的插件的JS Context至少是跟当前页面的HTML所在的Frame的Context可以互通。      

实际上，HTML内的JS的执行Context或者global，是以Frame为单位的。不同Frame内的JS无法直接相互访问对象和方法。在WebKit中，JavaScriptCore的对象模型如下图：

<center>
![Alt Text](/images/jsc_process.png)
</center> 


图中所示的Frame指的是HTML中的Frame所对应的WebCore C++ Frame。每个C++ Frame都会有唯一的ScriptController。这个ScriptController是WebKit通向JavaScriptCore的入口。一个ScriptController会有一个JSGlobalObject与之对应。当WebCore解析到一段内嵌或者外部的JS文件的时候，都会调用ScriptController的execute方法，在JavaScriptCore的VM中解析并执行这段JS代码。   


一个进程中，可以有多个WebView，或者多个Frame，多个Document，但是一个进程中，只有一个VM。我前面的一些文章有提到过，每一个JS对象，都会在JavaScriptCore中，有一个C++ JSCell的子类对象与之对应。这些JSCell，都是分配在JavaScriptCore的Heap中。在JavaScriptCore中，一个VM对象，对应一个Heap对象。

上面的图其实已经表达的很清楚了，在一个进程中，JavaScriptCore的VM和Heap，是进程级别的，所有的HTML的JS共享。而HTML内的JS又被Frame所限定。