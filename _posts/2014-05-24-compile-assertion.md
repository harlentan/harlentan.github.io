---
layout: post
title: Compile-time-assertion(编译断言)
tags: [C/C++]
keywords: Compile-time-assertion, 编译断言
description: 本文详细讲解了Compile-time-assertion(编译断言)的实现，以及Compile-time-assertion在WebKit中的运用
comments: true
share: true
---

&emsp;&emsp;编译断言技术现在被越来越多的使用。其目的在于让更多的错误在编译阶段被暴露，以便于更早的发现隐藏的bug。C++11中，提供了static_assert来用作编译断言，而对于还不支持C++11的地方，需要自己实现Compile-time-assertion。Compile-time-assertion的实现其实并不难，无非是定义一些非法的语句，让编译器报错即可。

&emsp;&emsp;WebKit中，被广泛使用的Compile-time-assertion是COMPILE_ASSERT，其实现如下：
{% highlight C++ %}
/* COMPILE_ASSERT */
#ifndef COMPILE_ASSERT
#if COMPILER_SUPPORTS(C_STATIC_ASSERT)
/* Unlike static_assert below, this also works in plain C code. */
#define COMPILE_ASSERT(exp, name) _Static_assert((exp), #name)
#elif COMPILER_SUPPORTS(CXX_STATIC_ASSERT)
#define COMPILE_ASSERT(exp, name) static_assert((exp), #name)
#else
#define COMPILE_ASSERT(exp, name) typedef int dummy##name [(exp) ? 1 : -1]
#endif
#endif
{% endhighlight %}
其中，`#define COMPILE_ASSERT(exp, name) typedef int dummy##name [(exp) ? 1 : -1]`，当exp是false的时候，dummy#name定义的数组长度为-1，从而编译器报错。

&emsp;&emsp;对于Compile-time-assertion的实现并不难，只需要构造一个编译错误即可。真正比较困难的是如何定义Compile-time-assertion条件, 该条件的值必须在编译阶段就已经知道。WebKit中，绝大多数COMPILE_ASSERT都用在使用sizeof比较大小，因为sizeof的值是在编译阶段即可确定的，另外就是用在静态常量值得比较，静态常量也是在编译阶段值必须确定。