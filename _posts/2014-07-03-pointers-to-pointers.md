---
layout: post
title: 重新认识二级指针(Pointers to Pointers)
tags: [C/C++, 程序员自我修养]
keywords: 二级指针, 双指针, 指针的指针, Pointers to Pointers, 程序员自我修养
description: 四年前(2010年)，我写了一篇关于我自己对于二级指针(Pointers to Pointers)的理解：《深入理解双指针》。这篇文章在网上一直存在着很大的争议，后面的评论也有很多质疑的声音。通过这几年我对C/C++更加深入的理解，我觉得有必要重新写一篇对于二级指针(双指针)的理解。
comments: true
share: true
---


四年前(2010年)，我写了一篇关于我自己对于二级指针(Pointers to Pointers)的理解：[《深入理解双指针》](http://blog.csdn.net/feiyinzilgd/article/details/5302369)。这篇文章在网上一直存在着很大的争议，后面的评论也有很多质疑的声音。通过这几年我对C/C++更加深入的理解，我觉得有必要重新写一篇对于二级指针(双指针)的理解。   

另外，本章中使用的程序是使用Linux的GCC编译出来的，所以汇编代码使用的是AT&T汇编指令，跟windows下使用Intel指令有所不同，详见[AT&T与Intel汇编比较](http://oss.org.cn/kernel-book/ch02/2.6.1.htm)。同时，由于我是用的是64位机器，为了方便讲解32位的程序以及防止编译器对代码的优化影响我们对问题的分析，本章所讲解的所有代码编译选项为：gcc -m32 -O0。

##概述

Pointers to Pointers：二级指针，我之前把它叫做双指针，比较专业的叫法是二级指针。二级指针是相对一级指针而言的。   
二级指针一般用于函数参数传递：  
      	
	addNode(Type** list);	

##C语言参数值传递

很多C语言书上，对于参数的值传递都讲解的不是很清楚。对于值传递的理解有助于理解我们理解二级指针。     

###普通变量的值传递

先看看一段代码：
{% highlight C linenos %}
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

void increase(int value)
{
    value = value + 1;
}

int main(int argc, char** argv)
{
    int count = 7;
    increase(count);
    printf("count = %d\n", count);

    return 0;
}
{% endhighlight %}
这段代码对应的汇编代码如下：
{% highlight objdump %}
080483e4 <increase>:
 80483e4:   55                      push   %ebp
 80483e5:   89 e5                   mov    %esp,%ebp
 80483e7:   83 45 08 01             addl   $0x1,0x8(%ebp)
 80483eb:   5d                      pop    %ebp
 80483ec:   c3                      ret    

080483ed <main>:
 80483ed:   55                      push   %ebp
 80483ee:   89 e5                   mov    %esp,%ebp
 80483f0:   83 e4 f0                and    $0xfffffff0,%esp
 80483f3:   83 ec 20                sub    $0x20,%esp
 80483f6:   c7 44 24 1c 07 00 00    movl   $0x7,0x1c(%esp)
 80483fd:   00  
 80483fe:   8b 44 24 1c             mov    0x1c(%esp),%eax
 8048402:   89 04 24                mov    %eax,(%esp)
 8048405:   e8 da ff ff ff          call   80483e4 <increase>
 //[...]
 {% endhighlight %}
这段代码执行的结果 count = 7。
我是用gdb调试，打印ESP和count的地址如下：

	(gdb) p $esp
	$2 = (void *) 0xffffd2b0
	(gdb) p &count
	$3 = (int *) 0xffffd2cc
main函数内部的汇编如下：
{% highlight gas %}
sub    $0x20,%esp #esp-0x20，栈向下生长0x20，用来存放局部变量
#在内存单元esp + 0x1c处存放7.
#即count,我上面打印的 $3 - #2 = 0x1c.
movl   $0x7,0x1c(%esp) 
  
mov    0x1c(%esp),%eax #将内存单元0x1c即count变量的值copy到EAX寄存器中
mov    %eax,(%esp) #copy count变量的内容到当前的ESP寄存器所指向的内存单元
call   80483e4 <increase> #调用increase函数
{% endhighlight %}
在我的机器上当前运行的ESP指针指向的内存单元是0xffffd2b0，栈向下生长了0x20，则当前栈桢(Stack Frame)的起始地址是0xffffd2b0到0xffffd2d0。count是局部变量，占用的是栈空间，上面gdb打印出来count的地址0xffffd2cc，正好落在main函数的栈桢内。  

有一点需要注意的是，在increase调用之前，count变量被copy了一份放在当前ESP所指向内存单元0xffffd2b0，这个count就是为了用来传递参数用的。   

接下来看看increase的汇编代码：
{% highlight gas %}
push   %ebp #ebp压栈，保护上一个栈桢
mov    %esp,%ebp #保护ESP
addl   $0x1,0x8(%ebp) #将copy出来的那个count变量+1
pop    %ebp
ret
{% endhighlight %}
increase的汇编代码比较简单，这里只需要解释下`addl   $0x1,0x8(%ebp)`。   

由前面一句`mov    %esp,%ebp`可以发现，此时EBP其实是指向栈顶。调用increase之前ESP是0xffffd2b0，由于调用increase需要将下一条IP指令压栈，则ESP = ESP - 0x04 = 0xffffd2ac。在进入increase之后，又执行了一句`push   %ebp`，ESP = 0xffffd2ac - 0x04 = 0xffffd2a8。那么此时栈顶就是0xffffd2a8，EBP的内容就是0xffffd2a8。`0x8(%ebp)`表示的是EBP + 0x8处的内存单元：0xffffd2a8 + 8 = 0xffffd2b0出的内存单元。

`addl   $0x1,0x8(%ebp)`这句汇编就是在内存单元0xffffd2b0处的内容加+1，最终将加一后的结果继续存放在0xffffd2b0处 。再回顾下，前面0xffffd2b0存放的内容：没错，就是copy出来的count。

看到这里，你会发现，在count传递到increase之后，一直都是在操作copy出来的那个count临时变量，而没有操作真正的count变量。可见，对于普通变量而言，参数的值传递就意味着只是简单的将变量copy了一份传递给函数，普通变量是无法改变外部原始变量的值。


###指针的值传递(一级指针)
还是先看代码：
{% highlight C linenos %}
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

void increase(int* ptr)
{
    *ptr = *ptr + 1;
}

int main(int argc, char** argv)
{
    int count = 7;
    increase(&count);
    printf("count = %d\n", count);
    return 0;
}
{% endhighlight C %}
这段代码对应的汇编代码如下：
{% highlight objdump %}
080483e4 <increase>:
 80483e4:   55                      push   %ebp
 80483e5:   89 e5                   mov    %esp,%ebp
 80483e7:   8b 45 08                mov    0x8(%ebp),%eax
 80483ea:   8b 00                   mov    (%eax),%eax
 80483ec:   8d 50 01                lea    0x1(%eax),%edx
 80483ef:   8b 45 08                mov    0x8(%ebp),%eax
 80483f2:   89 10                   mov    %edx,(%eax)
 80483f4:   5d                      pop    %ebp
 80483f5:   c3                      ret

080483f6 <main>:
 80483f6:   55                      push   %ebp
 80483f7:   89 e5                   mov    %esp,%ebp
 80483f9:   83 e4 f0                and    $0xfffffff0,%esp
 80483fc:   83 ec 20                sub    $0x20,%esp
 80483ff:   c7 44 24 1c 07 00 00    movl   $0x7,0x1c(%esp)
 8048406:   00
 8048407:   8d 44 24 1c             lea    0x1c(%esp),%eax
 804840b:   89 04 24                mov    %eax,(%esp)
 804840e:   e8 d1 ff ff ff          call   80483e4 <increase>
 // [...]
{% endhighlight %}
这段代码的执行结果是8。   
这段代码跟上一段代码的唯一区别是将count的地址传递给increase函数了。  

main函数的汇编代码
{% highlight gas %}
push   %ebp
mov    %esp,%ebp
and    $0xfffffff0,%esp
sub    $0x20,%esp
movl   $0x7,0x1c(%esp)

lea    0x1c(%esp),%eax #将count变量的地址赋值给EAX
mov    %eax,(%esp)
call   80483e4 <increase>
{% endhighlight %}
跟前面的main函数的唯一区别是`lea    0x1c(%esp),%eax`   


看懂这段代码首先要补习下lea指令。lea指令跟mov指令很相似，区别在于lea类似于C语言中的`&`取地址。那么lea操作也只是简单的针对地址做加法而已，而不会针对这个地址单元取操作数。  

那么这代码在调用increase函数之前，当前ESP所指向的内存单元的值是count变量的地址。而上一段代码在调用increase之前，当前ESP所指向的内存单元的值是count临时变量的值。   

我们再来看看increase函数的汇编代码
{% highlight gas %}
push   %ebp
mov    %esp,%ebp
mov    0x8(%ebp),%eax #前面已经讲过了
# 取出EAX所指向的内存单元的值赋值给EAX
# 也就是说执行此句话之后，EAX的内容是
# count变量的值，而不是地址。
mov    (%eax),%eax
lea    0x1(%eax),%edx #将EAX的内容加一，将加一后的结果存放到EDX
mov    0x8(%ebp),%eax #重新将count变量的地址赋值给EAX
#将EDX的内容存放到EAX所指向的内存单元
#就是将加一后的结果重新赋值给main函数里的count变量
mov    %edx,(%eax)
pop    %ebp
ret
{% endhighlight %}
理解这段汇编代码，需要记住一点，在调用increase之前，栈顶ESP所指向的内存单元的值是count变量的地址。之后，经过压栈IP，进入increase函数，再压栈EBP。则`0x8(%ebp)`，EBP + 0x8表示的就是在调用increase前，栈顶所指向的内存单元，里面存放的是count变量的地址。也就是说`mov    0x8(%ebp),%eax`之后，EAX的内容就是count变量的地址。紧接着`mov    (%eax),%eax`是现将EAX指向的内存单元的内容取出来存放到EAX中，此时EAX寄存器的内容已经不是地址了，而直接是count变量的值。然后对其做加一操作，存放到EDX当中。  

下面是最关键的两句话：
{% highlight gas %}
mov    0x8(%ebp),%eax
mov    %edx,(%eax)
{% endhighlight %}
由于EBP + 0x8里面放的是count变量的地址，`mov    0x8(%ebp),%eax`之后，EAX中存放的就是count变量的地址。   

EDX存放的是前面计算的结果，最后`mov    %edx,(%eax)`，将前面计算的结果重新存放到EAX所指向的内存单元，即重新给count变量赋值。


看到这里，你会发现，函数参数值传递，对于指针变量来说，也只是仅仅传递了一个内存地址，然后对这个内存地址进行操作。由于内存地址是进程级别的，所以，在函数内部 ，对地址所指向内容的修改，是可以带到函数外部的，是可以操作到函数外面的源变量的。


##二级指针
我们改造下上面的代码
{% highlight C linenos %}
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
void increase(int* ptr)
{
    *ptr = *ptr + 1;
    ptr = NULL;
}

int main(int argc, char** argv)
{
    int count = 7;  
    int* countPtr = &count;
    increase(countPtr);
    printf("count = %d\n", count);
    printf("countPtr = %p\n", countPtr);
    return 0;
}
{% endhighlight %}
运行结果，count = 8，而countPtr则不是NULL。  

运用前面的理论，其实很容易分析出问题。一级指针变量，也是一个普通变量，只不过这变量的值是一个内存单元的地址而已。countPtr在传递给increase之前，被copy到一个临时变量中，这个临时变量的值是一个地址，可以改变这个地址所在内存单元的值，但是无法改变外部的countPtr。   

从这个结果可以得出一个结论：一级指针作为参数传递，可以改变外部变量的值，即一级指针所指向的内容，但是却无法改变指针本身(如countPtr)。
  

有了上面的理解基础，其实对于理解二级指针已经很容易了。

对于指针操作，有两个概念：

* 引用：对应于C语言中的&取地址操作

[Reference](http://en.wikipedia.org/wiki/Reference_(computer_science))

* 解引用：在C语言中，对应于->操作。

[Dereference operator](http://en.wikipedia.org/wiki/Dereference_operator)

对于一个普通变量，引用操作，得到的是一级指针。一级指针传递到函数内部，虽然这个一级指针的值会copy一份到临时变量，但是这个临时变量的内容是一个指针，通过->解引用一个地址可以修改该地址所指向的内存单元的值。

<center>![Alt Text](/images/Pointer2Pointer.svg)</center>

对于一个一级指针，引用操作，得到一个二级指针。相反，对于一个二级指针解引用得到一级指针，对于一个一级指针解引用得到原始变量。一级指针和二级指针的值都是指向一个内存单元，一级指针指向的内存单元存放的是源变量的值，二级指针指向的内存单元存放的是一级指针的地址。

二级指针一般用在需要修改函数外部指针的情况。因为函数外部的指针变量，只有通过二级指针解引用得到外部指针变量在内存单元的地址，修改这个地址所指向的内容即可。


我们针对上面的代码继续做修改
{% highlight C linenos %}
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
void increase(int** ptr)
{
    **ptr = **ptr + 1;
    *ptr = NULL;
}

int main(int argc, char** argv)
{
    int count = 7;  
    int* countPtr = &count;
    increase(&countPtr);

    printf("count = %d\n", count);
    printf("countPtr = %p\n", countPtr);
    return 0;
}
{% endhighlight %}
这段代码，运行结果count = 8, countPtr = NULL;


##总结
首先，指针变量，它也是一个变量，在内存单元中也要占用内存空间。一级指针变量指向的内容是普通变量的值，二级指针变量指向的内容是一级指针变量的地址。


<span class="rating-foreground" style="width:90%"> 
   <meta itemprop="rating" content="4.5" /> 
</span>

