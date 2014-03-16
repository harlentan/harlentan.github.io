---
layout: post
title: 动态库的工作原理详解
tags: [blog]
keywords: 动态库的工作原理详解, SEO, 动态库，how shared libraries works，动态库加载, shared library, Android动态库
description: 详细讲述了动态库的加载以及工作原理
category: C/C++
comments: true
---
关于动态库的原理和加载过程，网上也有很多版本，但是基本都在讲解动态库的编译以及使用，很少能够有文章对动态库的加载以及工作原理进行深入的剖析和讲解。说来也很惭愧，在过去的工作中，没能彻底的去弄清楚动态库的工作原理。直到最近工作中听到一些关于动态库加载以及工作原理的一些错误的理论，一方面为了推翻该理论，另一方面，正好借此机会彻底弄清楚动态库的工作原理。  

<!--more-->   
##问题
在讲解动态库的工作原理之前，首先抛出几个问题，在讲解完之后，再回过头来分析问题。可能有些问题一看就是错的，但是我还是需要有正确的理论作为支撑来分析问题。    
1. 可以通过fork的方式，来降低使用同一个动态库的单独进程的内存占用。  
__问题背景__   
Android里面，可以通过adb shell showmap pid来查看某个进程的内存咱用其概况，中里面就列出来某个进程中某个动态库内存消耗，很多地方都称之为动态库的内存分摊。例如查看Android浏览器内存占用，里面将会有里边libwebcore.so内存占用 大小。所有就会有人觉得，动态库占用内存总量是一定的， 那么分摊的进程越多，最后分摊到单个进程上的内存占用就变得小了。所以可以通过这种技巧来降低内存占用。   
2. 如何优化动态库的内存占用？   

##Demo代码
下面的讲解会使用一个很简单的动态库以及使用动态库的程序来演示：    
greet.h   
{% highlight C linenos %}
// greet.h of libgreet.so
#ifndef GREET_H
#define GREET_H


extern void sayHi();



#endif
{% endhighlight %}  
greet.c    
{% highlight C linenos %}
// greet.c of libgreet.so
#include "greet.h"

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void sayHi() {
    printf("Hi I'am v1.0\n");
}
{% endhighlight %}
main.c
{% highlight C linenos %}
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
Linux/Unix的可执行文件以及动态库都是以ELF(Executable Linkage Format)存在的。在Linux下，可以使用readelf命令查看ELF文件，关于加载过程所需要的信息都在ELF文件头里面，可以用使用readefl filename -e来查看EFL文件所有的头。我们可以先来查看下main.c编译出来的test可执行文件的ELF头信息：   
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

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        08048154 000154 000013 00   A  0   0  1
  [ 2] .note.ABI-tag     NOTE            08048168 000168 000020 00   A  0   0  4
  [ 3] .note.gnu.build-i NOTE            08048188 000188 000024 00   A  0   0  4
  [ 4] .gnu.hash         GNU_HASH        080481ac 0001ac 00003c 04   A  5   0  4
  [ 5] .dynsym           DYNSYM          080481e8 0001e8 0000b0 10   A  6   1  4
  [ 6] .dynstr           STRTAB          08048298 000298 00008f 00   A  0   0  1
  [ 7] .gnu.version      VERSYM          08048328 000328 000016 02   A  5   0  2
  [ 8] .gnu.version_r    VERNEED         08048340 000340 000020 00   A  6   1  4
  [ 9] .rel.dyn          REL             08048360 000360 000008 08   A  5   0  4
  [10] .rel.plt          REL             08048368 000368 000018 08   A  5  12  4
  [11] .init             PROGBITS        08048380 000380 00002e 00  AX  0   0  4
  [12] .plt              PROGBITS        080483b0 0003b0 000040 04  AX  0   0 16
  [13] .text             PROGBITS        080483f0 0003f0 00017c 00  AX  0   0 16
  [14] .fini             PROGBITS        0804856c 00056c 00001a 00  AX  0   0  4
  [15] .rodata           PROGBITS        08048588 000588 000008 00   A  0   0  4
  [16] .eh_frame_hdr     PROGBITS        08048590 000590 000034 00   A  0   0  4
  [17] .eh_frame         PROGBITS        080485c4 0005c4 0000c4 00   A  0   0  4
  [18] .ctors            PROGBITS        08049f0c 000f0c 000008 00  WA  0   0  4
  [19] .dtors            PROGBITS        08049f14 000f14 000008 00  WA  0   0  4
  [20] .jcr              PROGBITS        08049f1c 000f1c 000004 00  WA  0   0  4
  [21] .dynamic          DYNAMIC         08049f20 000f20 0000d0 08  WA  6   0  4
  [22] .got              PROGBITS        08049ff0 000ff0 000004 04  WA  0   0  4
  [23] .got.plt          PROGBITS        08049ff4 000ff4 000018 04  WA  0   0  4
  [24] .data             PROGBITS        0804a00c 00100c 000008 00  WA  0   0  4
  [25] .bss              NOBITS          0804a014 001014 000008 00  WA  0   0  4
  [26] .comment          PROGBITS        00000000 001014 00002a 01  MS  0   0  1
  [27] .shstrtab         STRTAB          00000000 00103e 0000fc 00      0   0  1
  [28] .symtab           SYMTAB          00000000 0015ec 000410 10     29  45  4
  [29] .strtab           STRTAB          00000000 0019fc 0001f0 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)

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
 
 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rel.dyn .rel.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame 
   03     .ctors .dtors .jcr .dynamic .got .got.plt .data .bss 
   04     .dynamic 
   05     .note.ABI-tag .note.gnu.build-id 
   06     .eh_frame_hdr 
   07     
   08     .ctors .dtors .jcr .dynamic .got   
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
http://www.smilax.org/135/dsohowto.pdf
http://www.ibm.com/developerworks/library/l-shlibs/