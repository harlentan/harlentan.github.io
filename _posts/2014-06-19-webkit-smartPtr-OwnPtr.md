---
layout: post
title: 深入理解WebKit智能指针OwnPtr PassOwnPtr
tags: [WebKit]
keywords: 
comments: true
share: true
---

前面一篇[《深入理解WebKit智能指针RefPtr PassRefPtr》](http://www.fenesky.com/blog/2014/06/18/webkit-smartPtr-RefPtr.html)详细的讲解了WebKit的智能指针RefPtr和PassRefPtr。RefPtr和PassRefPtr是基于引用计数器的智能指针，在WebKit还有一种重要的智能指针OwnPtr和PassOwnPtr。从名称上可以看出，OwnPtr和PassOwnPtr是基于ownership的智能指针。

有了[《深入理解WebKit智能指针RefPtr PassRefPtr》](http://www.fenesky.com/blog/2014/06/18/webkit-smartPtr-RefPtr.html)这一章的基础，再来理解OwnPtr和PassOwnPtr会比较容易一点。智能指正本省其实是一个对象，那么作为对象，除了传递就是赋值了，还有就是销毁。智能指针的传递和赋值，就要涉及到ownership的传递了。

##OwnPtr

OwnPtr是基于ownership的智能指针。一个Raw pointer一旦交给OwnPtr管理，除非你使用leakPtr将这个Raw pointer从OwnPtr中leak出来，否则这个Raw Pointer的生命周期将有OwnPtr管理，这个管理就是控制这个Raw pointer的生命周期，甚至直接delete掉这个Raw pointer。OwnPtr::deleteOwnedPtr就是用来delete掉Raw pointer的。
<p/>
deleteOwnedPtr的实现如下：
{% highlight C++ linenos %}
template <typename T> inline void deleteOwnedPtr(T* ptr)
{
    typedef char known[sizeof(T) ? 1 : -1];
    if (sizeof(known))
        delete ptr;
}
{% endhighlight %}
可以看到，deleteOwnedPtr就是直接delete掉传进来的ptr。
<p/>
有三种操作可以导致deleteOwnedPtr被调用

<div style="text-align:center" markdown="1">
![Alt Text](/images/deleteOwnedPtr.svg)
</div>
其中，clear是使用者主动调用的。赋值操作，一方面是OwnPtr复制给OwnPtr，另一方面是PassOwnPtr赋值给OwnPtr。当一个新的Raw pointer需要传递给OwnPtr的时候，需要将新的Raw pointer的ownership转移给OwnPtr，此时OwnPtr一定是需要将旧的Raw pointer delete掉的，因为OwnPtr是旧的Raw pointer的owner。另外，当OwnPtr被析构的时候，也需要将OwnPtr所维护的Raw pointer delete掉，这个比较好理解，owner被析构掉了，owner附属的一些属性和指针也必须被delete掉。

##PassOwnPtr

跟PassRefPtr类似，PassOwnPtr存在的意义就是防止OwnPtr在相互传递的时候会导致Raw pointer频繁的被析构，会带来很多不必要的麻烦。PassOwnPtr向PassOwnPtr或者OwnPtr转化的时候，只是导致Raw pointer的ownership发生转移，而不会导致Raw pointer被delete掉。我们可以简单的理解为：PassOwnPtr就是为了OwnPtr的传递而存在的。

##OwnPtr和PassOwnPtr结合使用
先来看看OwnPtr、PassOwnPtr和Raw pointer的转化关系图

<div style="text-align:center" markdown="1">
![Alt Text](/images/ownPtr.svg)
</div>
Raw pointer想要转化成OwnPtr，只有一条途径：使用adoptPtr先转化成PassOwnPtr，再由PassOwnPtr转化成OwnPtr。
<p/>

同样，类似于RefPtr，OwnPtr和PassOwnPtr的get方法获取Raw pointer的方式比较危险，get方法仅仅只是返回Raw pointer而已，OwnPtr/PassOwnPtr不会有任何感知，很有可能会出现get获取的Raw pointer指向的内容在某一时刻被OwnPtr delete掉了。
<p/>
leakPtr的实现如下：
{% highlight C++ linenos %}
template<typename T> inline typename OwnPtr<T>::PtrType OwnPtr<T>::leakPtr()
{
    PtrType ptr = m_ptr;
    m_ptr = 0;
    return ptr;
}
{% endhighlight %}
leakPtr的行为是复位OwnPtr/PassOwnPtr使之失效，然后返回Raw pointer，相当于向OwnPtr/PassOwnPtr回收Raw pointer的管理权限。


##OwnPtr和PassOwnPtr使用规则

1. 构造OwnPtr必须使用adoptPtr。
2. 参数传递必须使用PassOwnPtr。
3. 类的成员只能使用OwnPtr而不能使用PassOwnPtr。
4. 对于一个new出来的对象如果其生命周期跟某个受某个类的控制，可以使用OwnPtr。