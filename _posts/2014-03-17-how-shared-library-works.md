---
layout: post
title: Linux动态库的工作原理详解
tags: [C/C++]
keywords: Linux动态库的工作原理详解, 动态库，how shared libraries works，动态库加载, shared library, Android动态库
description: 详细讲述了Linux动态库的加载以及工作原理
category: C/C++
comments: true
---
&emsp;&emsp;关于动态库的原理和加载过程，网上也有很多版本，但是基本都在讲解动态库的编译以及使用，很少能够有文章对动态库的加载以及工作原理进行深入的剖析和讲解。说来也很惭愧，在过去的工作中，没能彻底的去弄清楚动态库的工作原理。直到最近工作中听到一些关于动态库加载以及工作原理的一些错误的理论，一方面为了推翻该理论，另一方面，正好借此机会彻底弄清楚动态库的工作原理。     

##问题         
&emsp;&emsp;在讲解动态库的工作原理之前，首先抛出几个问题，在讲解完之后，再回过头来分析问题。可能有些问题一看就是错的，但是我还是需要有正确的理论作为支撑来分析问题。    
1. 可以通过fork的方式，来降低使用同一个动态库的单独进程的内存占用。  
__问题背景__   
&emsp;&emsp;Android里面，可以通过adb shell showmap pid来查看某个进程的内存咱用其概况，中里面就列出来某个进程中某个动态库内存消耗，很多地方都称之为动态库的内存分摊。例如查看Android浏览器内存占用，里面将会有里边libwebcore.so内存占用 大小。所有就会有人觉得，动态库占用内存总量是一定的， 那么分摊的进程越多，最后分摊到单个进程上的内存占用就变得小了。所以可以通过这种技巧来降低内存占用。   
2. 如何优化动态库的内存占用？   
如何优化动态库将在下一章节专门详细讲述。

##Demo代码
&emsp;&emsp;下面的讲解会使用一个很简单的动态库以及使用动态库的程序来演示：    
greet.h   
{% highlight C++ linenos %}
// greet.h of libgreet.so
#ifndef GREET_H
#define GREET_H


extern void sayHi();



#endif
{% endhighlight %}  
greet.c    
{% highlight C++ linenos %}
// greet.c of libgreet.so
#include "greet.h"

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void sayHi() {
    printf("Hi I'am v1.0\n");
}

static void __attribute__ ((constructor)) \
init_function(void)
{
    printf("Hello, Init Library!\n");
}

static void __attribute__((destructor)) \
fini_function (void)
{
    printf("Hello, Destruct Library!\n");
}

{% endhighlight %}
main.c
{% highlight C++ linenos %}
#include "greet.h"

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void sayHi() {
    printf("Hi I'am v1.0\n");
}
{% endhighlight %}
##动态库加载过程

###ELF基础

&emsp;&emsp;Linux/Unix的可执行文件以及动态库都是以ELF(Executable Linkage Format)存在的。在Linux下，可以使用readelf命令查看ELF文件，关于加载过程所需要的信息都在ELF文件头里面，可以用使用readefl filename -e来查看EFL文件所有的头。我们可以先来查看下main.c编译出来的test可执行文件的ELF头信息：   
{% highlight text %}
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x80483f0
  Start of program headers:          52 (bytes into file)
  Start of section headers:          4412 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         9
  Size of section headers:           40 (bytes)
  Number of section headers:         30
  Section header string table index: 27


Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x08048034 0x08048034 0x00120 0x00120 R E 0x4
  INTERP         0x000154 0x08048154 0x08048154 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /lib/ld-linux.so.2]
  LOAD           0x000000 0x08048000 0x08048000 0x00688 0x00688 R E 0x1000
  LOAD           0x000f0c 0x08049f0c 0x08049f0c 0x00108 0x00110 RW  0x1000
  DYNAMIC        0x000f20 0x08049f20 0x08049f20 0x000d0 0x000d0 RW  0x4
  NOTE           0x000168 0x08048168 0x08048168 0x00044 0x00044 R   0x4
  GNU_EH_FRAME   0x000590 0x08048590 0x08048590 0x00034 0x00034 R   0x4
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4
  GNU_RELRO      0x000f0c 0x08049f0c 0x08049f0c 0x000f4 0x000f4 R   0x1
  
{% endhighlight %}
对于一个exe应用程序启动过程中，通过系统调用exec族函数来替换掉当前进程的内容为要加载的应用程序，从而进入内核，内核需要找到程序执行的入口，那么这个入口是由Program Headers来提供的。此时，内核只知道进程的起始地址，是无法找到这个程序的执行入口的，这是，就需要ELF Header来辅助了。根据约定，ELF Header是被加载到offset为0的进程空间地址上， 也就是说ELF Header的地址是已知的，ELF Header中定义了Program Headers的偏移量。   
{% highlight text %}   
Start of program headers:          52 (bytes into file)
Start of section headers:          4412 (bytes into file)
Flags:                             0x0
Size of this header:               52 (bytes)
{% endhighlight %}
ELF Header的片段中，Start of program headers, Size of this header就可以定位Program Headers的地址。
##Dynamic Linker的加载
&emsp;&emsp;在Kernel找到程序的Program Header之后，就开始执行该程序的指令。我们可以看到：
{% highlight text %}
Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x08048034 0x08048034 0x00120 0x00120 R E 0x4
  INTERP         0x000154 0x08048154 0x08048154 0x00013 0x00013 R   0x1
      [Requesting program interpreter: /lib/ld-linux.so.2]
{% endhighlight %}
INIERP, 指定了程序的Dynamic Linker。 程序启动过程中，另外一个重要的工作就是启动Dynamic Linker。这个Dynamic Linker其实做了三件事情:   

1. 加载程序所依赖的库
2. 重新分配应用程序和依赖库的内存地址。
3. 初始化应用程序    

&emsp;&emsp;加载依赖的库，这一步比较简单。我们可以通过readelf -d 查看程序Dynamic Sections. 上面的Demo的Dynamic Sections如下：
{% highlight text %}
  Tag        Type                         Name/Value
 0x00000001 (NEEDED)                     Shared library: [libgreet.so]
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
 0x0000000c (INIT)                       0x8048380
{% endhighlight %}
+ 加载依赖
这里记录了程序所依赖的其他动态库，只需要递归的查找这个区的数据就可以获取所有依赖库的列表，然后挨个加载即可。
+ 重新分配地址
重新分配地址，分为了两种情况：

    1. 基于相对地址的重分配

&emsp;&emsp;实际上，程序内部的函数地址以及全局全局变量的地址在编译阶段就已经知道了。我这里说的地，指的是相对地址。当Dynamic Linker运行的时候，Linker是知道进程的起始地址的，所以对于相对地址的重分配而言，比较简单，只需要使用进程的地址+相对地址即可。

    2. 基于符号的地址重分配

&emsp;&emsp;这一步是重新分配地址过程最复杂最耗时一部。在程序编译阶段，当遇到使用动态库中的变量和函数的，其实是不知道该变量和函数的地址的。这个时候，Linker就会查找符号然后放到PLT(Procedule Linkage Table)里面，这个表在程序运行过程中是不需要更改的，所以是Read-Only.这样，程序运行过程中，就可以知道需要使用的函数和变量地址了。当然，程序并不是直接查这个表的。而是通修改一个叫GOT(Global Offset Table).也就是说每次程序使用动态库里的函数和变量的时候，就会向GOT去取，这个操作在编译阶段就已经形成。在运行时，只需要改变GOT对应的地址即可。

    3. 初始化

&emsp;&emsp;注意看我的Demo greet.c中，定义了init_函数，这个函数就是在这个阶段被调用的。也就是说在程序加载动态库完成之后，在执行应用程序任何代码之前，会调用动态库的初始化函数。当然，我还定了动态库的析购函数， 是在整个应用程序结束之后卸载动态库的时候被调用的。
  
  
##结束
&emsp;&emsp;到此为止，我想大家应该对动态库的加载以及原理有了一些了解。我们在回过头来看看一开始我列出的问题：   
可以通过fork的方式，来降低使用同一个动态库的单独进程的内存占用。我想通过上面的分析，答案应该是确定的：不能。因为即便是fork以及所谓的分摊，其实对于动态库的加载来说，应该说是按需分配，也就是上面讲的Dynamic Linker加载阶段，查找使用了哪些，然后放到PLT中。即便是fork出来的，Dynamic Linker需要重新加载，重新构建PLT。 除非不调用exec族函数替换进程内容。
