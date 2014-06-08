---
layout: post
title: JavaScriptCore非Heap内存的管理
tags: [JavaScriptCore]
keywords: JavaScriptCore内存占用不释放, 深入浅出JavaScriptCore, JavaScriptCore内存泄露,JavaScriptCore内存管理
description: 结合JavaScriptCore源码分析讲解JavaScriptCore非Heap内存的管理以及如何快速降低JavaScriptCore的内存占用。
category: C/C++, JavaScriptCore
comments: true
share: true
---

由于JavaScript是为Web而设计的，使得JavaScript携带数据能力非常有限，而随着HTML5以及WebAPP的发展，对于JavaScript携带数据的能力要求越来越高。JavaScriptCor内置对象几乎不携带数据，除了Number, String之类以外。为了满足WebApp的数据能力，我们需要对JavaScriptCore进行扩展。也就意味着我们需要在JavaScriptCore里为JavaScript new/malloc一块内存来存放数据。这部分内存是没办法在JavaScriptCore的Heap上分配的，也即是我们所说的非Heap内存。对于非Heap内存如果管理不好，一方面会导致内存泄露，另一方面会导致JavaScriptCore的内存占用持续增高无法释放。

##Heap内存的分配   
所有的JavaScript对象在JavaScriptCore中创建的时候，都是在Heap上分配的内存。创建方式比较固定：
{% highlight C++ %}
ObjectType* number = new (NotNull, allocateCell<ObjectType>(vm.heap)) ObjectType(vm, structure);
{% endhighlight %}
通过`replacement new`方式，让allocateCell在Heap分配一块内存来存放Object。其中ObjectType必须直接或者间接继承与JSCell。只要是在Heap上分配的内存，JavaScriptCor管理起来非常方方便，如果JavaScript对象之间的引用关系处理好了，基本不会导致内存持久点增大。

##非Heap内存的分配与管理   
首先需要指出的是，在JavaScriptCore不能使用C++的new或者malloc的方式来申请一块内存，这样做虽然JavaScriptCore编译不报错，而且有时候运行也没问题，但是如果你尝试这样做了，你会发现程序内存占用不仅会越来越大，而且还会经常莫名其妙的crash。C++的new/malloc会破坏JavaScriptCore的GC行为，GC的过程中也会不固定的在某些地方crash。    

由于JavaScriptCore的Heap内存管理机制非常完善，可以考虑比较小的buffer(小于10K)在Heap上分配空间。JavaScriptCore提供了如下接口：
{% highlight C++ linenos %}
CheckedBoolean tryAllocateStorage(size_t size, void**);
CheckedBoolean tryReallocateStorage(void**, size_t, size_t);
{% endhighlight %}
需要注意的是，size的字节数必须是8的倍数，否则会导致Heap其他对象内存分配错乱，使得内存互踩，程序很容易Crash。    

对于大内存的申请，就没办法在Heap上分配了，即便是使用`tryAllocateStorage`，程序也会crash。也就意味着大内存必须在Heap外分配空间，而且也不能使用new/malloc，只能使用WTF::fastMalloc/WTF::fastFree, WTF::fastNew/WTF::fastDelete等WTF的内存申请方式。JavaScriptCore的GC触发时机是由当前申请的Heap内存大小与当前Heap大小来决定的。如果在Heap之外分配空间，JavaScriptCore是无法感知到这不比分内存占用的存在的。所以，即便是在Heap外分配的内存快爆了，GC也无动于衷，因为它不知道这部分内存的存在。JavaScriptCore在设计的时候，考虑到了这一点，提供了接口用来告知JavaScriptCore，我们在Heap之外申请了内存：    
{% highlight C++ linenos %}
//C++ API
void reportExtraMemoryCost(size_t cost);

//C API
void JSReportExtraMemoryCost(JSContextRef ctx, size_t size);
{% endhighlight %}
通过这种方式让JavaScriptCore感知Heap之外这部分内存的存在，迫使JavaScriptCore尽量早的去GC Heap上没有被引用的对象。       

内存申请之后，如何被释放你的？我们这样做，一定是将申请的buffer指针跟当前的JavaScript对象绑定了，我们希望在此JavaScript对象被释放的时候，通过WTF::fastFree掉这部分内存。最合理的是将fastFree buffer的操作放到JavaScript对应的C++对象的析构函数里面来做。但是你会发现，所有的JavaScriptCore的C+ 对象(直接或者间接继承与JSCell)的析构函数都无法被自动调用，除非你在代码中显示的调用析构函数。因为JavaScriptCore在Heap上分配内存的时候，是通过`replacement new`方式来在JavaScriptCore mmap出来的一大块内存(每一块至少是64K)中分配一小块，当JavaScript对象没有引用之后，JavaScriptCore会直接sweep这部分内存然后让给其他新申请的对象使用。所以，JavaScriptCore的C++对象的析构函数永远不会被调用。幸好JavaScriptCore提供了一个机制：
{% highlight C++ linenos %}
static const bool needsDestruction = true;
static void destroy(JSCell*);
{% endhighlight %}       
如果希望JavaScriptCore在将要析构对象的时候通知你，需要在自己的类中定义和实现上面的语句。JavaScriptCore在Sweep对象的时候，会判断`needsDescription`的值，如果是true，就会调用该对象的destroy函数，这样，我们就可以在destroy实现中显示的调用类的析构函数。`needsDescription`在JSCell中被定义为false。所以继承者需要根据自己的需求来重新定义。       

目前的JavaScriptCore的实现除了依赖于WTF意外，还高度依赖于WebKit的消息循环机制。GC的过程中，其实是fire一个timer，让WebKit的消息循环来调度GC。如果想要把JavaScriptCore从WebKit中独立出来使用，还需要实现和部分GC fire的代码。否则，你会发现GC不工作，内存暴涨。
