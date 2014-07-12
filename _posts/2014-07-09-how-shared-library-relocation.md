---
layout: post
title: Linux动态库原理(二)重定位
tags: [C/C++, 程序员自我修养]
keywords: Linux动态库原理, 动态库重定向, share library relocation
description: 前面一章《Linux动态库工作原理详解》比较简单浅显的对 Linux 的工作原理进行了阐述，今天打算从 Linux 动态库在加载过程中符号的重定位(Relocation)的角度，更加深入的讲解 Linux 动态库的工作原理。
category: C/C++
comments: true
share: true
---

##前言

前面一章[《Linux动态库工作原理详解》](http://www.fenesky.com/blog/2014/03/17/how-shared-library-works.html)比较简单浅显的对 Linux 的工作原理进行了阐述，今天打算从 Linux 动态库在加载过程中符号的重定位(Relocation)的角度，更加深入的讲解 Linux 动态库的工作原理。  

在1980s SunOS 将动态库引入到 UNIX，后来又将 ELF(Executable and Linkable) 格式引入到了 UNIX。在现在看来，动态库的出现主要解决了一系列的问题。内存问题，内核不需要为每个进程都加载一份动态库，除数据区以外，所有进程共享一个动态库。可扩展性，在程序不用重启的情况下，动态的加载所需要的动态库，可实现对程序的扩展。另外动态库还解决了程序动态更新的问题，通过对动态库版本的控制，可以完美的对程序接口进行更行升级。  

   

在正式开始讲解 Linux 动态库工作原理的符号重定向之前，先给大家介绍下在本文中用到的几个重要的工具：  

* readelf     
 一个可以查看ELF格式文件的强大工具。
* objdump    
 可以将ELF格式的文件转化成汇编指令。   
* gdb     
 在本文分析问题的过程中，需要gdb进行调试来查看寄存器信息。  

##动态库编译链接

先贴出来本文要讲解的 libincrease.so 的源码
{% highlight C++ linenos %}
// increase.c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int count = 7;

void increase()
{
    count = count + 1;
}
{% endhighlight %}

任何程序，想要使用count或者increase，都必须知道这些变量和函数在内存中的地址。最终反应到汇编代码里面，需要访问任何变量和函数，都需要使用mov指令，将某个内存单元的地址传送到一个寄存器中。

由于动态库是在程序运行过程中被载入到进程的地址空间的，在动态库被编译和链接期间，动态库内所有的符号都是用的是相对地址。不仅如此，任何程序都可以使用任何数量的动态库，且同一个动态库在不同程序中被加载到进程空间的顺序也不一样。也就意味着编译和链接阶段是无法知道要使用的变量和函数在进程空间的地址。

为了解决程序运行过程中能够准确的找到动态库中的符号，在Linux中，有两种方案：

1. 加载过程中重定位
2. 使用Position Independent Code(PIC)

重定位和PIC是通过在编译动态库的时候使用-shared或者-fPIC来控制的。本文主要讲解动态库加载过程中的重定位。   

libincrease.so的编译命令如下：

	gcc -m32 -g -c increase.c  -o increase.o
	gcc -m32 -shared -o libincrease.so increase.o
为了方便对问题的分析，我这里使用了-m32来强行编译32位的程序。  

我们先使用readelf -h来查看libincrease.so的ELF文件头信息：
	
	ELF Header:
	Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
	Class:                             ELF32
	Data:                              2's complement, little endian
	Version:                           1 (current)
	OS/ABI:                            UNIX - System V
	ABI Version:                       0
	Type:                              DYN (Shared object file)
	Machine:                           Intel 80386
	Version:                           0x1
	Entry point address:               0x380
	Start of program headers:          52 (bytes into file)
	Start of section headers:          4984 (bytes into file)
	Flags:                             0x0
	Size of this header:               52 (bytes)
	Size of program headers:           32 (bytes)
	Number of program headers:         7
	Size of section headers:           40 (bytes)
	Number of section headers:         33
	Section header string table index: 30
根据[ELF Specification](http://www.skyfree.org/linux/references/ELF_Format.pdf)的解释。
ELF头部有16个字节，其中第一个字节7f是固定的魔数，后面的 45，4c，46分别是'E', 'L', 'F'对应的ASIC码值。后面三个字节依次标示当前的ELF支持的系统位数(32位/64位)，大小端以及版本号。其中 Entry point address 0x380 表示进程开始执行的地方。

##变量的重定位
使用命令:

	readelf --segments libincrease.so

得到的动态库程序头部信息如下

	Program Headers:
	Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
	LOAD           0x000000 0x00000000 0x00000000 0x00520 0x00520 R E 0x1000
	LOAD           0x000f0c 0x00001f0c 0x00001f0c 0x00104 0x0010c RW  0x1000
	DYNAMIC        0x000f20 0x00001f20 0x00001f20 0x000c8 0x000c8 RW  0x4
	NOTE           0x000114 0x00000114 0x00000114 0x00024 0x00024 R   0x4
	GNU_EH_FRAME   0x0004a4 0x000004a4 0x000004a4 0x0001c 0x0001c R   0x4
	GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0x4
	GNU_RELRO      0x000f0c 0x00001f0c 0x00001f0c 0x000f4 0x000f4 R   0x1

	Section to Segment mapping:
	Segment Sections...
	00     .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version 
	.gnu.version_r .rel.dyn .rel.plt .init .plt .text .fini .eh_frame_hdr .eh_frame 
	01     .ctors .dtors .jcr .dynamic .got .got.plt .data .bss 
	02     .dynamic 
	03     .note.gnu.build-id 
	04     .eh_frame_hdr 
	05     
	06     .ctors .dtors .jcr .dynamic .got

从这个头部信息，我们可以看到有两个LOAD，其中第一个LOAD被标记为RE，只读可执行，说明是代码段。第二个LOAD被标记为RW，可读写，说明是数据段。数据段从0x00001f0c(VirtAddr)开始，长度是0x00104(FileSiz)。

下面的Section to Segment mapping是上面的程序头部每一个地址段对应的一个map，01对应于上面的第二个LOAD。从01可以看到.data以及.bss都在0x00001f0c开始的数据段内。

我们在使用命令：

	readelf -r libincrease.so
看下动态重定位区的数据：

	Relocation section '.rel.dyn' at offset 0x2dc contains 6 entries:
 	Offset     Info    Type            Sym.Value  Sym. Name
	00002008  00000008 R_386_RELATIVE   
	00000440  00000701 R_386_32          0000200c   count
	00000448  00000701 R_386_32          0000200c   count
	00001fe8  00000106 R_386_GLOB_DAT    00000000   __cxa_finalize
	00001fec  00000206 R_386_GLOB_DAT    00000000   __gmon_start__
	00001ff0  00000306 R_386_GLOB_DAT    00000000   _Jv_RegisterClasses
.rel.dyn表示动态重定位区。动态库在加载过程中的重定位就是使用.rel.dyn重定位数据。在这里可以看到count出现了2次，是因为count在increase函数中，需要被使用2次。所以，在这个动态库中，有两个地方需要用到count的内存单元的地址。后面的Sym.value表示count这个符号在整个动态库中的偏移量为0x200c。

通过以下命令

	objdump -d -Mintel libincrease.so

dump出来的increase函数汇编代码如下：

{% highlight objdump %}
0000043c <increase>:
 43c:	55                   	push   ebp
 43d:	89 e5                	mov    ebp,esp
 43f:	a1 00 00 00 00       	mov    eax,ds:0x0
 444:	83 c0 01             	add    eax,0x1
 447:	a3 00 00 00 00       	mov    ds:0x0,eax
 44c:	5d                   	pop    ebp
 44d:	c3                   	ret    
 44e:	90                   	nop
 44f:	90                   	nop
{% endhighlight %}
通过这段汇编代码，可以很明显的函数mov    eax,ds:0x0是在将count传送到eax中去。但是此时count的地址是多少呢？

我们再回头看看上面的动态重定位区的数据。第一个count对应的是在increase第一次出现的count。Type R\_386\_32，通过查看[ELF specification]()，发现R\_386\_32操作的意思就是将符号的地址存放到Offset对应的地方去。那么在这的意思就是，将count的真实地址存放到偏移量位0x440的地方去。我们在看看偏移量位0x440得地方有什么。我们可以看到，increase汇编代码中第一次取count的代码
	
	43f:	a1 00 00 00 00       	mov    eax,ds:0x0
其中a1是mov指令的编码，该编码占一个字节，后面紧跟着是mov的操作数存放在0x440的地方。看到这里，就明白了，动态重定位区的第一个count给出的信息是将count的真实地址存放到偏移量位0x440的地方，这样就完成了第一个count的重定位。

为了验证这个分析，我们来动态调试一番。为了获取动态库在内存中载入的起始地址，我们使用了dl\_iterate\_phdr。该函数可以在程序运行过程中获取载入该进程的所有动态库的信息。

{% highlight C++ linenos %}
#define _GNU_SOURCE
#include <link.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

static int header_handler(struct dl_phdr_info* info, size_t size, void* data)
{
    printf("name=%s (%d segments) address=%p\n",
            info->dlpi_name, info->dlpi_phnum, (void*)info->dlpi_addr);
    for (int j = 0; j < info->dlpi_phnum; j++) {
         printf("\t\t header %2d: address=%10p\n", j,
             (void*) (info->dlpi_addr + info->dlpi_phdr[j].p_vaddr));
         printf("\t\t\t type=%u, flags=0x%X\n",
                 info->dlpi_phdr[j].p_type, info->dlpi_phdr[j].p_flags);
    }   
    printf("\n");
    return 0;
}

extern void increase();

int main(int argc, char** argv)
{
    dl_iterate_phdr(header_handler, NULL);
    increase();

    return 0;
}
{% endhighlight %}
编译命令如下：

	gcc -m32 -std=gnu99  -g main.c  -L. -lincrease -o main
运行之前，需要将libincrease.so的路径export到LD\_LIBRARY\_PATH当中才能让程序正常运行。

使用gdb调试，得到的libincrease.so在进程地址空间信息如下：
	
	name=libincrease.so (7 segments) address=0xf7fd6000
		 header  0: address=0xf7fd6000
			 type=1, flags=0x5
		 header  1: address=0xf7fd7f0c
			 type=1, flags=0x6
		 header  2: address=0xf7fd7f20
			 type=2, flags=0x6
		 header  3: address=0xf7fd6114
			 type=4, flags=0x4
		 header  4: address=0xf7fd64a4
			 type=1685382480, flags=0x4
		 header  5: address=0xf7fd6000
			 type=1685382481, flags=0x6
		 header  6: address=0xf7fd7f0c
			 type=1685382482, flags=0x4
我们可以得到的信息是libincrease.so从进程的地址空间0xf7fd6000开始。根据之前的理论，在动态库加载过程中重定位的时候，将动态库Offset位0x440处的内存单元内容替换成了count的真实地址了。而之前我们已经分析出，count变量在动态库内部的偏移量是0x200c。所以，此时count在当前进程地址空间的地址应该是 0xf7fd6000 + 0x200c = 0xf7fd800c。不仅如此，在动态库偏移量位0x440处的内容也应该是count的地址，即0xf7fd800c。所以，在0xf7fd6000 + 0x440 = 0xf7fd6440处应该是0xf7fd800c。

OK。接下来我就要在gdb中打印count的地址和内存地址0xf7fd6440处的内容了：

	(gdb) print &count
	$7 = (int *) 0xf7fd800c
	(gdb) x /4xb 0xf7fd6440
	0xf7fd6440 <increase+4>:	0x0c	0x80	0xfd	0xf7
Cool，奇迹出现了。跟我们与其的一模一样！需要注意的是，在我打印0xf7fd6440处的内容的时候，我是用了x /4xb 0xf7fd6440，关于如何查看指定内存的gdb命令请参考[Examining memory](http://www.delorie.com/gnu/docs/gdb/gdb_56.html)。

##函数的重定位

上面分析了关于动态库中变量的重定位。现在再来看看动态库中的函数，在动态库加载的过程中是如何完成重定位的。
为了验证动态库内部函数的重定位，我们需要对increase.c进行改造
{% highlight C++ linenos %}
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

int count = 7;

void increase()
{
    count = count + 1;
}

void doincrease(int n)
{
    count += n;
    increase();
}
{% endhighlight %}

	objdump -d -Mintel libincrease.so

dump出来的doincrease函数的汇编代码：
{% highlight objdump %}
0000047e <doincrease>:
 47e:   55                      push   ebp 
 47f:   89 e5                   mov    ebp,esp
 481:   a1 00 00 00 00          mov    eax,ds:0x0
 486:   03 45 08                add    eax,DWORD PTR [ebp+0x8]
 489:   a3 00 00 00 00          mov    ds:0x0,eax
 48e:   e8 fc ff ff ff          call   48f <doincrease+0x11>
 493:   5d                      pop    ebp 
 494:   c3                      ret    
 495:   90                      nop 
{% endhighlight %}
从汇编代码可以看出在48e:   e8 fc ff ff ff          call   48f <doincrease+0x11>处，会调用increase函数。

我们还是需要去动态重定向区查看信息
	
	Relocation section '.rel.dyn' at offset 0x2f4 contains 9 entries:
	Offset     Info    Type            Sym.Value  Sym. Name
	00002008  00000008 R_386_RELATIVE   
	00000470  00000701 R_386_32          0000200c   count
	00000478  00000701 R_386_32          0000200c   count
	00000482  00000701 R_386_32          0000200c   count
	0000048a  00000701 R_386_32          0000200c   count
	0000048f  00000602 R_386_PC32        0000046c   increase
	00001fe8  00000106 R_386_GLOB_DAT    00000000   __cxa_finalize
	00001fec  00000206 R_386_GLOB_DAT    00000000   __gmon_start__
	00001ff0  00000306 R_386_GLOB_DAT    00000000   _Jv_RegisterClasses

那么我们需要弄清楚R\_386\_PC32，根据[ELF Specification](http://www.skyfree.org/linux/references/ELF_Format.pdf)，R\_386\_PC32指的是将Offset内存单元的内容加上Sym.value重定位之后符号的真实地址再减去Offset重定位之后的r\_offset，并将结果存放到r\_offset对应的内存单元。  

那么对于我们分析函数increase调用的重定位，我们只需要知道libincrease.so在进程空间载入的地址即可，其他的都可以进行计算。
同样，我们还是使用dl\_iterate\_phdr来获取动态库在进程地址空间的信息：
{% highlight C++ linenos %}
#define _GNU_SOURCE
#include <link.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

static int header_handler(struct dl_phdr_info* info, size_t size, void* data)
{
    printf("name=%s (%d segments) address=%p\n",
            info->dlpi_name, info->dlpi_phnum, (void*)info->dlpi_addr);
    for (int j = 0; j < info->dlpi_phnum; j++) {
         printf("\t\t header %2d: address=%10p\n", j,
             (void*) (info->dlpi_addr + info->dlpi_phdr[j].p_vaddr));
         printf("\t\t\t type=%u, flags=0x%X\n",
                 info->dlpi_phdr[j].p_type, info->dlpi_phdr[j].p_flags);
    }   
    printf("\n");
    return 0;
}

extern void doincrease(int n); 

int main(int argc, char** argv)
{
    dl_iterate_phdr(header_handler, NULL);
    doincrease(6);

    return 0;
}
{% endhighlight %}
编译命令跟之前的一样。使用gdb调试，查看libincrease.so在内存中载入的地址
	
	name=libincrease.so (7 segments) address=0xf7fd6000
		 header  0: address=0xf7fd6000
			 type=1, flags=0x5
		 header  1: address=0xf7fd7f0c
			 type=1, flags=0x6
		 header  2: address=0xf7fd7f20
			 type=2, flags=0x6
		 header  3: address=0xf7fd6114
			 type=4, flags=0x4
		 header  4: address=0xf7fd64f4
			 type=1685382480, flags=0x4
		 header  5: address=0xf7fd6000
			 type=1685382481, flags=0x6
		 header  6: address=0xf7fd7f0c
			 type=1685382482, flags=0x4

我们可以看到libincrease.so载入地址是0xf7fd6000，根据rel.dyn中increase的sym.value值可以得出，在进程中increase函数的真实地址是0xf7fd6000 + 0x46c = 0xf7fd646c。  

根据上面.rel.dyn的信息，对于increase的重定位，我们需要将Offset 0x48f处的内容加上increase重定位后的真实地址，再减去Offset重行为后的r\_offset的内容，将结果写回到r\_offset标示的内存单元内。   

通过上面objdump出来的doincrease中0x48f内存单元的内容是0xfffffffc。对于Offset 0x48f重定位之后r\_offset的值是0xf7fd6000 + 0x48f = 0xf7fd648f。那么.rel.dyn对于重定位increase函数的计算结果是0xf7fd646c + 0xfffffffc - 0xf7fd648f = 0xffffffd9。然后将0xffffffd9写回到r\_offset 0xf7fd648f处。那么这一系列的重定向之后最终得到的结果应该是在进程地址空间0xf7fd648f内存单元的内容变为0xffffffd9。

OK。我们通过gdb打印结果如下：
	
	(gdb) print increase
	$2 = {void ()} 0xf7fd646c <increase>
	(gdb) disas /r doincrease
	Dump of assembler code for function doincrease:
	0xf7fd647e <+0>:	55	push   %ebp
	0xf7fd647f <+1>:	89 e5	mov    %esp,%ebp
	0xf7fd6481 <+3>:	a1 0c 80 fd f7	mov    0xf7fd800c,%eax
	0xf7fd6486 <+8>:	03 45 08	add    0x8(%ebp),%eax
	0xf7fd6489 <+11>:	a3 0c 80 fd f7	mov    %eax,0xf7fd800c
	0xf7fd648e <+16>:	e8 d9 ff ff ff	call   0xf7fd646c <increase>
	0xf7fd6493 <+21>:	5d	pop    %ebp
	0xf7fd6494 <+22>:	c3	ret    
	End of assembler dump.

通过gdb打印出increase在进程中的地址是0xf7fd646c。

	disas /r doincrease

这条gdb命令是打印运行期间doincrease的汇编代码。

我们发现在0xf7fd648f内存单元处的内容是0xffffffd9。是不是很cool。所有的结果跟我们之前计算的结果一样。doincrease中call指令对应的code是e8，后面紧跟着是参数0xffffffd9，也即是我们刚刚计算出来的值，是一个相对地址。那么真正开始准备跳转到increase的时候，IP内的地址是0xf7fd6493，程序运行期间，找到increase的地址是0xf7fd6493 + 0xffffffd9 = 0xf7fd646c。回头看看，0xf7fd646c处是什么？是increase函数。

##结束

到此为止，我们已经将动态库加载期间，动态库内的变量和函数的重定位都分析完毕。对于shared方式重定位技术，对于变量重定位过程中就是直接修改所有使用变量的地方，将其直接修改为变量的真实地址。对于函数重定位，就是将调用函数的地方，将其内容修改为当前地址到需要调用的函数的Offset。