---
layout: post
title: C++对象内存分布详解
tags: [JavaScriptCore, C++]
keywords: C++对象内存分布详解, this指针传递， this指针, Linux C++对象内存分布, C++对象内存分布原理
description: 结合汇编语言，讲解C++对象的内存分布情况以及应用场景
comments: true
share: true
---

很多资料或者书上，关于C++对象的大小，定义为C++类（包括父类）的所有非静态成员变量的大小，然后再加上虚函数表的指针。关于这一理解，需要了解C++对象的内存分布， 以及类中所有成员的存储位置，才能对C++对象的大小进行解释。在JavaScriptCore的Heap内存管理中，有很多地方利用了C++对象内存分布的特性，通过this指针加偏移量来直接操作对象的属性。


在讲解指针，我们首先来看一段非常简单的代码：

{% highlight C++ linenos %}
#include <unistd.h>
#include <stdlib.h>

using namespace std;


class A { 
public:
    static int mem1;
    void init()
    {   
        this->mem2 = 8;
        this->mem3 = 16; 
        A::mem1 = 32; 
    }   

private:
    int mem2;
    int mem3;
};

int A::mem1 = 0;

int main(int argc, char** argv)
{
    A a;
    a.init();

    return 0;
}
{% endhighlight linenos %}
代码在Android上编译之后通过objdump命令，dump出来的主要汇编代码如下：
{% highlight objdump %}
00000378 <_ZN1A4initEv>:
 378:   b082        sub sp, #8
 37a:   9001        str r0, [sp, #4] 
 37c:   9b01        ldr r3, [sp, #4] 
 37e:   f04f 0208   mov.w   r2, #8
 382:   601a        str r2, [r3, #0] 
 384:   9b01        ldr r3, [sp, #4] 
 386:   f04f 0210   mov.w   r2, #16 
 38a:   605a        str r2, [r3, #4] 
 38c:   4b03        ldr r3, [pc, #12]   ; (39c <_ZN1A4initEv+0x24>)
 38e:   447b        add r3, pc
 390:   f04f 0220   mov.w   r2, #32 
 394:   601a        str r2, [r3, #0] 
 396:   b002        add sp, #8
 398:   4770        bx  lr  
 39a:   bf00        nop 
 39c:   00001c72    .word   0x00001c72

000003a0 <main>:
 3a0:   b500        push    {lr}
 3a2:   b085        sub sp, #20 
 3a4:   9001        str r0, [sp, #4] 
 3a6:   9100        str r1, [sp, #0] 
 3a8:   ab02        add r3, sp, #8
 3aa:   4618        mov r0, r3
 3ac:   f7ff ffe4   bl  378 <_ZN1A4initEv>
 3b0:   f04f 0300   mov.w   r3, #0
 3b4:   4618        mov r0, r3
 3b6:   b005        add sp, #20 
 3b8:   bd00        pop {pc}
 3ba:   bf00        nop
{% endhighlight %}
从汇编代码可以很明显的发现，main函数中`3ac:   f7ff ffe4   bl  378 <_ZN1A4initEv>`这条指令，跳转到378，也就是上面`00000378 <_ZN1A4initEv>`，A::init函数的实现。从这里就可以看出，成员函数地址在编译阶段就已经确定，成员函数并不是存储在对象内部。

然后分析00000378 <_ZN1A4initEv>的实现：
sub sp, 38是给init函数分配栈空间，str r0, [sp, #4]，此时r0内部存放的是参数this指针，讲this写到init函数栈空间偏移为4的地方。紧接着， ldr r3, [sp, #4]，就是讲刚刚在栈上写入的this指针取出来存放到r3上，此时r3里存放的就是this指针。在回头看看上面的C++代码，init的实现是：分别给mem1, mem2和mem3赋值为：8， 16， 32。init的汇编代码里，比较明显的是mov.w，依次将8，16，32存放到r2中, 另外，根据上面的分析，此时r3中存放的是this指针，那么382:   601a    str r2, [r3, #0]是将8写入到mem2中, 38a:   605a        str r2, [r3, #4]是在将16写入到mem3中，可以说明，mem2存放位置是对象所在内存偏移量为0的地方，mem3则存放在对象内存偏移量为4的地方。从38c:   4b03        ldr r3, [pc, #12]   ; (39c <_ZN1A4initEv+0x24>)开始，r3发生了变化，不在是this指针了， 此时r3存放的是pc+12之后的地址，就是在取得我们C++代码中的静态成员变量mem1的地址00001c72。可见，静态成员变量没有存放在对象内部。

对于C++类来说，我个人理解，无论有没有继承，只要是在没有虚函数的情况下，对象的大小都可以依据上面的分析来解释对象的大小和内存分布。在有虚函数的情况下，稍微有些不同。

{% highlight C++ linenos %}
#include <unistd.h>
#include <stdlib.h>

using namespace std;


class A { 
public:
    static int mem1;
    virtual void sayHi() { this->mem2 = 64; }
    void init()
    {   
        this->mem2 = 8;
        this->mem3 = 16; 
        A::mem1 = 32; 
    }   

    virtual ~A() {}

private:
    int mem2;
    int mem3;
};

int A::mem1 = 0;

int main(int argc, char** argv)
{
    A a;
    a.init();
    a.sayHi();

    return 0;
 }
{% endhighlight %}

{% highlight objdump %}
000003d8 <_ZN1A4initEv>:
 3d8:   b082        sub sp, #8
 3da:   9001        str r0, [sp, #4] 
 3dc:   9b01        ldr r3, [sp, #4] 
 3de:   f04f 0208   mov.w   r2, #8
 3e2:   605a        str r2, [r3, #4] 
 3e4:   9b01        ldr r3, [sp, #4] 
 3e6:   f04f 0210   mov.w   r2, #16 
 3ea:   609a        str r2, [r3, #8] 
 3ec:   4b03        ldr r3, [pc, #12]   ; (3fc <_ZN1A4initEv+0x24>)
 3ee:   447b        add r3, pc
 3f0:   f04f 0220   mov.w   r2, #32 
 3f4:   601a        str r2, [r3, #0] 
 3f6:   b002        add sp, #8
 3f8:   4770        bx  lr  
 3fa:   bf00        nop 
 3fc:   00001c12    .word   0x00001c12
 {% endhighlight %}
 我们可以发现，init函数的实现流程几乎一样，唯一的差别是给mem2赋值的时候，r3中的this指针要偏移4个字节，给mem3赋值的时候，r3中的this指针需要偏移8个字节。也就是说成员变量的偏移整体移动了4个字节。由于当有虚函数的时候，会有一个虚函数表的指针，该指针固定的存放在对象其实地址偏移量为0的地方。

 上面分析的是无继承的情况下，在有继承的情况下，