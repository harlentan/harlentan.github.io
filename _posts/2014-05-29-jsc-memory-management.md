---
layout: post
title: JavaScriptCore Heap内存管理
tags: [JavaScriptCore]
keywords: JavaScriptCore内存管理, JavaScriptCore Heap, JavaScriptCore Heap内存申请, JavaScriptCore GC 垃圾回收
description: 深入JavaScriptCore原理讲解JavaScriptCore内存管理。JavaScriptCore内存管理是JavaScriptCore里面最为复杂的一个模块。从JavaScript对象在JavaScriptCore Heap申请内存开始，到GC 回收JavaScriptCore Heap内存。
comments: true
share: true
---

前面的一篇[《JavaScriptCore非Heap内存的管理》](http://www.fenesky.com/blog/2014/04/26/javascript-memory-occupation.html)主要是从JavaScriptCore非Heap内存的角度讲解JavaScriptCore是如何管理这部分内存的。对于JavaScriptCore的内存管理来讲，最重要的部分是JavaScriptCore对于Heap内存的管理。JavaScript里面所有对象的内存都是在JavaScriptCore的Heap里面分配的，JavaScriptCore的Heap内存管理直接影响到JavaScript的性能。今天就跟大家分享下我对于JavaScriptCore Heap内存管理的理解。

##JavaScriptCore Heap基本概念
JavaScriptCore的Heap，是由JavaScriptCore通过mmap向系统映射的一块内存区域，并由JavaScriptCore独立的进行管理。JavaScriptCore中，Heap对象只存在与VM中。而对于WebKit来说，WebKit中的JavaScriptCore Heap是一个单例，一个WebKit进程中只有一个JavaScriptCore Heap，所以HTML中的JavaScript如果写的有问题，例如循环引用之类的导致的内存泄漏，只要WebKit进程还在，即使页面关闭之后，JavaScript引起的内存泄漏也不会释放。JavaScriptCore Heap在创建的时候，有`largeHeapSize`和`smallHeapSize`两种类型的Heap，其中largeHeapSize的GC阀值是32M，smallHeapSize的GC阀值是1M。并不是这个GC阀值越小越好，阀值太小，反而使得JavaScriptCore频繁的GC，性能降低，另外，阀值太小会使得JavaScriptCore内存申请经常失败。

##JavaScriptCore Heap内存管理原理

<div style="text-align:center" markdown="1">
![Alt Text](/images/heap.svg)
</div>

在JavaScriptCore的Heap中，有三个重要的概念：`Region`，`Block`，`Cell`。他们的size大小以此递减。

* Region

Region，区域，对应于`class Region`。JavaScriptCore Heap内存是分区的，一个区是mmap映射的64K。当一个Region的内存用完之后，再继续创建下一个Region，也就是64K。

* Block

Block, 块，对应于`MarkedBlock`。Block是在Region内部分配的。一个Block的大小是64K，也就是说一个Region内部只有一个Block。

* Cell

Cell，就是用来存放JSObject对象的。在JavaScriptCore中，没有专门的一个类来表示Cell。但是Cell的概念却对于内存分配和回收起着重要作用。Cell是JavaScriptCore Heap上内存分配的最小单位。Cell是在Block上分配的。Cell虽然没有特定的类来表示，但是Cell是由大小的， Cell的大小是根据一个特定逻辑计算出来的。另外，在Block创建的时候，就需要确定该Block内部Cell的大小。也就是说对于同一个Block来说，其内部的Cell大小是一样的，使得Cell的大小要大于或者等于JSObject，如果大于JSObject的size，必定将会有一定空间的浪费。

###Subspace
Subspace是用来申请Block的。JavaScriptCore中，有三种Subspace：

* normalDestructorSpace：

Block内部Cell中存放的JSObject被析构的时候需要调用JSObject的destroy的Space

* immortalStructureDestructorSpace: 

同normalDestructorSpace类似，尚未发现跟normalDestructorSpace不同之处，但是还是有差异，我之前使用JavaScriptCore的时候发现我写的JSObject如果是分配在immortalStructureDestructorSpace，会crash，在其他space则不会。这一点我还在最跟踪。

* normalSpace: 

前两者之外的space都叫normalSpace。

那么也就意味着上面三种Subspace会分配三种不同类型的MarkedSpace。JSOBject是由权利决定自己落在那种类型的MarkedBlock的。通过在自定义的JSObject中定义如下变量:
{% highlight C++ %}
static const bool needsDestruction = true;
static const bool hasImmortalStructure = true;
{% endhighlight %}
在不定义的情况下，使用的JSCell中的needsDestruction = false, hasImmortalStructure = false, 则JSOBject被分配到normalSpace当中。

Subspace的定义如下：
{% highlight C++ %}
struct Subspace {
	// preciseCount = 16
    FixedArray<MarkedAllocator, preciseCount> preciseAllocators;
    // impreciseCount = 128
    FixedArray<MarkedAllocator, impreciseCount> impreciseAllocators;
    MarkedAllocator largeAllocator;
};
{% endhighlight %}
其中，MarkedAllocator就是实际用来给JSObject申请内存空间的。
当需要给一个JSObject申请内存空间的时候，首先需要选择合适的MarkedAllocator：
{% highlight C++ %}
inline MarkedAllocator& MarkedSpace::allocatorFor(size_t bytes)
{
    ASSERT(bytes);
    if (bytes <= preciseCutoff)
        return m_normalSpace.preciseAllocators[(bytes - 1) / preciseStep];
    if (bytes <= impreciseCutoff)
        return m_normalSpace.impreciseAllocators[(bytes - 1) / impreciseStep];
    return m_normalSpace.largeAllocator;
}
{% endhighlight %}
其中bytes是JSObject的size, preciseCutoff是常量128， impreciseCutoff是常量32K。如果一个JSObject的size <= 128的，会在preciseAllocators数组中按照allocatorFor的算法选择一个MarkedAllocator来使用。我们可以推算下，size 在[121 - 128] 之间的JSObject使用的是preciseAllocators[15]，size在[113 - 120]之间的JSObject，使用的是preciseAllocators[14]，以此类推。
讲到这里，可能有点晕了，看看下面的图：

<div style="text-align:center" markdown="1">
![Alt Text](/images/subspace.svg)
</div>



###DeadBlock

{% highlight C++ %}
class DeadBlock : public HeapBlock<DeadBlock> {
public:
    DeadBlock(Region*);
};
{% endhighlight %}
DeadBlock结构非常简单，DeadBlock其实我们可以理解为每一块将要分配给MarkedBlock的内存头部的一个指针。上面已经提到，要分配Block，首先要分配Region，Region内部在对MarkedBlock的内存进行划分，在划分之前，需要在每块内存的头部构造一个DeadBlock。当需要分配MarkedBlock的时候，只需要找到任意一个DeadBlock的指针，并在该指针指向的内存构造一个MarkedBlock即可。当MarkedBlock需要被回收的时候，只需要在该MarkedBlock指针指向的地方重新构造一个DeadBlock。


###Cell size的计算
在这些内存分配结构都讲完之后，再回头看看前面说到的，每一块Block内部，Cell的size都是相同的。cell size的计算逻辑如下：
{% highlight C++ %}
MarkedSpace::MarkedSpace(Heap* heap)
    : m_heap(heap)
{
    for (size_t cellSize = preciseStep
        ;cellSize <= preciseCutoff
        ;cellSize += preciseStep) {
        allocatorFor(cellSize).init(heap
            ,this
            ,cellSize, MarkedBlock::None);
        normalDestructorAllocatorFor(cellSize).init(heap
            ,this
            ,cellSize
            ,MarkedBlock::Normal);
        immortalStructureDestructorAllocatorFor(cellSize).init(heap
            ,this
            ,cellSize
            ,MarkedBlock::ImmortalStructure);
    }

    for (size_t cellSize = impreciseStep
        ;cellSize <= impreciseCutoff
        ;cellSize += impreciseStep) {
        allocatorFor(cellSize).init(heap
            ,this
            ,cellSize
            ,MarkedBlock::None);
        normalDestructorAllocatorFor(cellSize).init(heap
            ,this
            ,cellSize
            ,MarkedBlock::Normal);
        immortalStructureDestructorAllocatorFor(cellSize).init(heap
            ,this
            ,cellSize
            ,MarkedBlock::ImmortalStructure);
    }

    m_normalSpace.largeAllocator.init(heap
        ,this
        ,0
        ,MarkedBlock::None);
    m_normalDestructorSpace.largeAllocator.init(heap
        ,this
        ,0
        ,MarkedBlock::Normal);
    m_immortalStructureDestructorSpace.largeAllocator.init(heap
        ,this
        ,0
        ,MarkedBlock::ImmortalStructure);
}

{% endhighlight %}
cellSize的计算其实也是分为precise和imprecise的。其中，preciseStep = 8, preciseCutoff = 128. impreciseStep = 256, impreciseCutoff = 32 K. 拿precise来说，由preciseAllocator分配出来MarkedBlock的cell size都是8的倍数，由impreciseAllocator分配出来MarkedBlock的cell size都是256的倍数。


###BlockAllocator
上面讲到的MarkedAllocator只是去申请给JSObject分配空间的Allocator，而真正去分配MarkedBlock的是BlockAllocator，MarkedAllocator会把一些属性例如cellsize传递给BlockAllocator，让BlockAllocator去分配MarkedBlock。BlockAllocator的分配工作是在主线程中完成的。除此之外，BlockAllocator创建另一个线程，专门用来回收Region所管理的64K内存。一个Region内部有可能有多个MarkedBlock，当Region内部所有的MarkedBlock都被析构掉之后，Heap认为这个Region为空，那么此时就可以向系统归还这部分内存，这些操作是在BlockAllocator的非主线程中操作的。


###FreeList

上面讲到，Cell是存放JSObject的，FreeList是MarkedBlock中空闲cell 指针的一个列表。每当需要给JSObject分配类存的时候，如果在Region和Block都够用的情况下，只需要在符合当前JSObject size的MarkedBlock上的FreeList上取出一个指针就是指向一个Cell，返给该指针，就可以在这个指针所指向的内存上构造一个JSObject了。

讲到这里，Heap内存管理的进本流程都已经讲完了，再来看下面这个图，就比较轻松了：

<div style="text-align:center" markdown="1">
![jsc-mem](/images/jsc-mem.svg)
</div>
Heap的内存分配，首先会构造一个Region，这个Region是mmap出来的64k内存空间。紧接着，对Region进行划分为若干个MarkedBlock(64K)大小的区域，通常情况下其实一个Region内部只有一个MarkedBlock。但是此时还没有在该内存里面构造MarkedBlock。划分之后，会在每块MarkedBlock管理的64K内存的头部构造一个DeadBlock。在需要的时候，只需要返回DeadBlock的头指针，然后再该指针指向的内存地址上构造一个MarkedBlock。之后，需要对MarkedBlock内部Cell进行划分，这部分内存空间的大小是64K减去MarkedBlock本身size后的空间。然后根据上面的Cell Size的计算逻辑，将一个MarkedBlock管理的内存划分为若干个Cell，然后将空闲的Cell的FreeList返回。

###主要类图：

<div style="text-align:center" markdown="1">
![jsc-mem](/images/jsc-mem-class.svg)
</div>
