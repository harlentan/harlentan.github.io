---
layout: post
title: WebKit/WebCore中JavaScript在JavaScriptCore上的执行
tags: [JavaScriptCore, WebKit]
keywords: WebKit/WebCore中JavaScript在JavaScriptCore上的执行, JavaScriptCore, WebKit/WebCore JS执行, WebKit/WebCore JavaScriptCore, WebKit运行JavaScript
description: 详细讲解了JavaScript在WebKit/WebCore的JavaScriptCore上的执行流程和机制
category: C/C++
comments: true
---

&emsp;&emsp;WebKit/WebCore中JavaScript在JavaScriptCore上的执行主要从WebKit/WebCore的层面详细讲解了JavaScript在WebKit内部的执行流程和机制。写前端的同学都知道，HTML页面内的javascript是无法操作其他页面以及其他Frame内部的元素和对象的。不仅如此，HTML内部的JavaScript不同的写法也会影响HTML的加载和运行效率，而JavaScript的执行效率对于移动端的WebApp影响尤其明显。那么，WebKit究竟是如何对JavaScript的执行环境进行隔离的呢？JavaScript的写法又是如何影响HTML的加载和执行效率的呢？本文将针对这些问题进行详细的分析和讲解。  

<!--more-->   

##前言  
&emsp;&emsp;Android 4.2及其以下的版本的WebKit都支持JS引擎在V8和JavaScriptCore之间进行切换。为了尽量保持JS引擎在WebCore Bindings层面接口保持一致，V8同样也定义了很多跟JavaScriptCore类似的类和接口。但是由于V8和JavaScriptCore在很理念的差异，使得JavaScript在WebKit/WebCore上的执行环境差异比较大。另外在JS执行效率上，最新版本的JavaScriptCore的性能绝对不输V8。JavaScriptCore把大部分对于JavaScript基础运算以及执行的优化都放到汇编级别，特别是对于执行比较复杂的JavaScript程序。不仅如此，JavaScriptCore还提供了一套框架以便于很容易的替换JavaScriptCore的一些比较核心的基础功能，例如Malloc。令人吃惊的是，JavaScriptCore上运行JavaScript的某些运算，效率甚至超过了C/C++。    

##Global对象   
&emsp;&emsp;在JavaScriptCore中，任何全局的属性和方法都被挂在JSGlobalObject上，例如Number, Boolean, Array。在WebCore里面，JavaScript执行环境的Global Object就是window对象，也就是JSDOMWindow。这部分代码我们可以在WebKit/Source/Webcore/Bindings下面找到，JSDOMWindow.h, JSDOMWindow.cpp文件时动态生成的文件。这也就是为什么window对象上挂在的属性和方法可以直接使用而不必通过window.xxx的方式调用的原因。对于任何HTML页面，只要有window对象，就可以执行JavaScript。   

##JavaScript执行环境    

&emsp;&emsp;在WebCore中，JavaScript执行环境其实是以Frame为基本单位的。Frame将JavaScript执行的控制权交给了ScriptController(对应ScriptController.cpp)来处理。window对象就是ScriptController通过JSDOMWindodwShell创建出来的。每一个Frame都会有唯一的一个ScriptController，一个ScriptController通常情况下只管理一个windowShell(JSDOMWindow)对象，windowShell从命名上就可以看出，只是window对象的封装而已。ScriptController也会有对应多个windowShell的情况，例如再有Plugin的时候，也会给Plugin创建一个windowShell对象，并交给ScriptController管理。window之间相隔离，也就是说在一个window对象的内部无法访问另一个window对象的方法和属性。当需要执行一段JavaScript的时候，WebCore其实即使将该段JavaScript交给对应的JSGlobalObject去evaluate()，这里的JSGlobalObject也就是前面提到的window对象。  

##HTML内部的JavaScript对象内存管理    

&emsp;&emsp;我之前一直有一个误区，误认为只要浏览器的HTML页面被关闭，或者通过刷新或者重定向或者同一个tab里面加载另一个HTML页面，那么无论如何之前显示的HTML内部的JavaScript对象所占用的内存就应该被GC掉并回收了。最近在跟踪一个奇怪的问题，当前页面都已经被刷新或者加载其他HTML了，上一个HTML页面内的JavaScript对象占用的内存竟然没有被释放。后来通过查看代码才发现，WebCore中，使用的是JavaScriptCore的一个全局VM对象，改VM对象永远不会被析构，就算进程结束了，也不会调用改VM对象的析构函数。WebCore的VM对象定义如下：
{% highlight C++ %}
// JSDOMWindodwBase.cpp
VM* JSDOMWindowBase::commonVM()
{
    ASSERT(isMainThread());

    static VM* vm = 0;
    if (!vm) {
        ScriptController::initializeThreading();
        vm = VM::createLeaked(LargeHeap).leakRef();
#ifndef NDEBUG
        vm->exclusiveThread = currentThread();
#endif
        initNormalWorldClientData(vm);
    }

    return vm;
}
{% endhighlight %}   
JavaScriptCore的Heap是由VM管理的。也就是说，所有的HTML的JavaScript对象都是在一个全局的Heap上申请的内存。HTML页面被析构掉之后，仅仅只是DOM Tree, Render Tree等这些结构被析构掉了。如果HTML页面中某些JavaScript对象有循环引用导致window对象无法被GC，就很有肯那个导致整个页面的JavaScript对象都无法被正常GC掉，就会造成虽然HTML页面已经消失了，但是该HTML页面内的JavaScript对象还在占用内存且无法被释放。  

##JavaScript执行
&emsp;&emsp;其实对于JavaScript的执行流程相对于前面讲到，比较简单那。script标签的async和defer决定了JavaScript的执行时机，在HTML文档解析的过程中，遇到script的async和defer，豆浆推迟JavaScript的加载时机。如果什么都不加，默认是阻塞执行，也就是解析HTML的过程中，遇到script标签，会停止HTML的解析，直到JavaScript被执行完。一开始不是很理解，作为一款非常优秀的排版渲染器来说，怎么会允许JavaScript阻塞页面的显示呢？后来查看W3C中关于HTML解析过程的解释是，JavaScript执行过程中，有可能会对DOM节点进行操作，这些操作将会导致DOM树重建，消耗比较大，所以要将这种影响减低到最小，也就是执行JavaScript的时候，停止解析HTML。JavaScript脚本最终会被传给ScriptController，然后交给JSMainThreadExecState， JSMainThreadExecState才是真正最终去执行JavaScript的地方。

##WebCore中JavaScript相关的主要类图
![Alt text](/images/webkit_jsc.PNG)   
从上图可以看到，当有需要执行一段JavaScript的时候，只需要找打对应的Frame，script()->evaluate()来执行JavaScript。evaluate的过程中，需要一个windowShell，也就是通过windowShell()来获取，如果没有，就需要创建一个JSDOMWindowShell, 在JSDOMWindowShell创建好之后，setWindow来创建JSDOMWindow，也就是window对象。evaluate最终执行JavaScript并不是由ScriptController直接操作的，而是转交给JSMainThreadExecState来执行的。
