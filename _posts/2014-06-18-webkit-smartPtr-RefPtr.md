---
layout: post
title: 深入理解WebKit智能指针RefPtr PassRefPtr
tags: [WebKit]
keywords: WebKit智能指针, WTF智能指针, PassRefPtr, RefPtr, WebKit RefPtr, WebKit PassRefPtr, WebKit RefPtr PassRefPtr
description: 2005年以前，WebKit都使用的一套基于`ref count`引用计数的智能指针。2005年，Apple发现这套智能指针在WeKit中导致ref和deref使用混乱，一方面是出现很多内存泄漏。另一方面是效率很低，因为智能指针经常会需要被赋值以及作为参数传递，在这种情况下，每当有赋值或者参数传递的操作，都会导致`ref count`被加一或者减一，效率比较低。WebKit的工程师一直都在尝试着寻找一种易用且高效的智能指针的解决方案。后来，WebKit的工程师Maciej Stachowiak受到C++的[auto ptr](http://www.cplusplus.com/reference/memory/auto_ptr/)的启发：智能指针的传递，是指针ownership的传递，发明了RefPtr和PassRefPtr的引用计数智能指针。
comments: true
share: true
---

##WebKit智能指针历史

2005年以前，WebKit都使用的一套基于`ref count`引用计数的智能指针。2005年，Apple发现这套智能指针在WeKit中导致ref和deref使用混乱，一方面是出现很多内存泄漏。另一方面是效率很低，因为智能指针经常会需要被赋值以及作为参数传递，在这种情况下，每当有赋值或者参数传递的操作，都会导致`ref count`被加一或者减一，效率比较低。WebKit的工程师一直都在尝试着寻找一种易用且高效的智能指针的解决方案。后来，WebKit的工程师Maciej Stachowiak受到C++的[auto ptr](http://www.cplusplus.com/reference/memory/auto_ptr/)的启发：智能指针的传递，是指针ownership的传递，发明了RefPtr和PassRefPtr的引用计数智能指针。


##RefPtr

RefPtr主要就是实现了ref和deref的机制。当对RefPtr与RefPtr进行赋值的时候，Raw pointer的ownership会发生转移
{% highlight C++ linenos %}
template<typename T> template<typename U> 
inline RefPtr<T>& RefPtr<T>::operator=(const RefPtr<U>& o)
{
    T* optr = o.get();
    refIfNotNull(optr);
    T* ptr = m_ptr;
    m_ptr = optr;
    derefIfNotNull(ptr);
    return *this;
}
{% endhighlight %}
首先将新的Raw pointer通过`o.get()`取出来然后对其引用计数加一并保存在`m_ptr`当中，而对`m_ptr`的值的引用计数减一。从而实现了Raw pointer的ownership的转移：对旧的`m_ptr`进行引用计数减一，在`m_ptr`中保存新的Raw pointer。

单独使用RefPtr写程序的时候，会出现类似下面的代码：
{% highlight C++ linenos %}
// example, not preferred style;
// should use RefCounted and adoptRef (see below)
 
RefPtr<Node> createSpecialNode()
{
    RefPtr<Node> a = new Node;
    a->setSpecial(true);
    return a;
}

RefPtr<Node> b = createSpecialNode();
{% endhighlight %}
根据前面的一个文章[《Return Value Optimization(RVO)返回值优化》](http://www.fenesky.com/blog/2014/06/17/RVO.html)
中所提到的，在编译器不对代码进行优化的情况下，上面这段代码会导致拷贝拷贝构造函数被调用很多次。即使编译器做了返回值优化，同样也会导致`ref count`被频繁的加一减一，没有达到提高效率的目的

##PassRefPtr

PassRefPtr相对于RefPtr的一个最大优势是：PassRefPtr在向PassRefPtr或者RefPtr进行赋值和转化的时候，只发生ownership的转移，不对`ref count`进行操作。

PassRefPtr转化成RefPtr的代码如下：
{% highlight C++ linenos %}
template<typename T> template<typename U> 
inline RefPtr<T>::RefPtr(const PassRefPtr<U>& o)
	: m_ptr(o.leakRef())
{
}
{% endhighlight %}
我们再来看看`PassRefPtr::leakRef`的实现：
{% highlight C++ linenos %}
template<typename T> inline T* PassRefPtr<T>::leakRef() const
{
    T* ptr = m_ptr;
    m_ptr = 0;
    return ptr;
}
{% endhighlight %}
直接将Raw pointer返回并将当前PassRefPtr保存的Raw pointer赋为0，使得当前PassRerPtr失效。    
<p/>
那么再来看看PassRefPtr转化成RefPtr的操作，不难发现，其实这个操作只做了一件事情：Raw pointer的ownership转移到RefPtr中了，并没有对`ref count`进行操作。

如果我们单独使用PassRefPtr写程序
{% highlight C++ linenos %}
// warning, will dereference a null pointer and will not work
 
static RefPtr<Ring> g_oneRingToRuleThemAll;

void finish(PassRefPtr<Ring> ring)
{
    g_oneRingToRuleThemAll = ring;
    ...
    ring->wear();
}
{% endhighlight %}
由于`g_oneRingToRuleThemAll = ring`这个语句做了PassRefPtr的赋值操作，此时`ring`这个PassRefPtr已经失效，因为该赋值操作会将`ring`内部保存的Raw pointer值赋为0，也就是说，`ring->wear()`会出现空指针异常。

##RefPtr和PassRefPtr结合使用

有了上面的分析，你会发现，为了达到易用和高效的目的，必须将RefPtr和PassRefPtr结合起来使用。
<p/>
我们先来一个RefPtr、PassRefPtr和Raw Pointer之间的转换图

<div style="text-align:center" markdown="1">
![Alt Text](/images/passPtr.svg)
</div>

###RefPtr向PassRefPtr转化

`RefPtr::release`的实现如下：
{% highlight C++ %}
PassRefPtr<T> release()
{
	PassRefPtr<T> tmp = adoptRef(m_ptr);
	m_ptr = 0;
	return tmp;
}
{% endhighlight %}
先用当前RefPtr的`m_ptr` Raw pointer构造一个PassRefPtr，然后将`m_ptr`赋为0，使得当前RefPtr被复位失效。使得Raw pointer的ownership发生了转移，从原来的RefPtr转移到新的PassRefPtr中。


###PassRefPtr向Raw pointer转化

直接通过上面提到的`leakRef`，返货Raw pointer，并把当前PassRefPtr复位使之失效。

###RefPtr向Raw pointer转化

RefPtr要转化为Raw pointer，有2条途径：

1. 直接使用get
2. 先通过release转化为PassRefPtr，然后再将PassRefPtr转化为Raw pointer。

<p/>
我们先来看看`RefPtr::get`的实现
{% highlight C++ %}
T* get() const { return m_ptr; }
{% endhighlight %}
直接将Raw pointer返回，既没有发生ownership的转移，也没有对`ref count`进行操作。这是一个很危险的操作，意味着凡是通过get方式返回的Raw pointer，在任何情况下都不能对它进行`ref count`和delete操作，因为此时RefPtr是这个Raw pointer的owner，这些操作必须由RefPtr去完成。
<p/>
当然，你还会发现，PassRefPtr也可以get方法转化成Raw pointer，原理跟RefPtr::get的相同，也是一个比较危险的操作，返回的Raw pointer的行为受限。

<p/>
RefPtr转化成PassRefPtr，再由PassRefPtr转化成Raw pointer 是一个比较安全的操作，因为发生了ownership的转移，转化为PassRefPtr之后，再转化成Raw pointer就比较容易了。

###PassRefPtr向RefPtr转化

这一操作，上面也提到了，只发生Raw pointer的ownership的转移，不对引用计数进行操作，原PassRefPtr被复位使之失效。

##Ref Count的保存

上面讲了很多关于`ref count`引用计数的操作，这里我们需要明确一点，RefPtr和PassRefPtr并不保存Ref Count。Ref Count引用计数器是存放在Raw pointer的。RefPtr和PassRefPtr对Raw pointer进行ref和deref操作，也就意味着Raw pointer必须有ref和deref成员函数。WebKit中，一般需要使用RefPtr和PassRefPtr的类都继承于RefCounted或者ThreadSafeRefCounted。
<p/>
RefCounted或者ThreadSafeRefCounted都继承自RefCountedBase。RefCountedBase中有个private成员变量就是`unsigned m_refCount`，并公开了`ref()`和`derefBase()`接口。其中，`ref()`就是对`m_refCount`做加一操作。`derefBase()`则是对`m_refCount`做减一操作并返回true 或者false，来标示是否需要delete this指针。

<p/>
看下RefCounted::deref的实现
{% highlight C++ %}
void deref()
{
    if (derefBase())
        delete static_cast<T*>(this);
}
{% endhighlight %}
通过derefBase返回的值来确定是否需要delete this指针。因为derefBase会对`m_refCount`做减一操作，减一之后结果如果是0，说明这个对象已经没有引用了，`derefBase`就会返回true，则可以delete this指针。


##对Raw pointer生命周期的影响
通过上面的分析，我们可以发现，RefPtr和PassRefPtr仅仅只是调用Raw pointer的`ref()`和`deref()`来改变Raw pointer的`m_refCount`。Raw pointer自己根据当前`m_refCount`的值来决定是否需要delete this。
<p/>
也就是说RefPtr和PassRefPtr并不直接参与Raw pointer的delete操作。这一点跟OwnPtr和PassOwnPtr有所不同，关于OwnPtr和PassOwnPtr后面会专门写一篇文章来讲解。


##RefPtr和PassRefPtr的使用规则

1. 参数的传递要使用PassRefPtr。
2. 成员变量不能是PassRefPtr。
3. 函数内部的局部变量要使用RefPtr。
4. 函数返回值要使用PassRefPtr。
5. 对于一个新create出来的对象，应该尽早的转化成RefPtr。
5. 继承与`RefCounted`的对象应使用`adoptRef`转化成PassRefPtr。
