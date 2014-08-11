---
layout: post
title: C语言字符串常量的存储
tags: [程序员自我修养, C/C++]
keywords: C语言常量, C语言字符串常量, C语言字符串常量的存储
description: 网络上也有很多关于字符串常量的讨论和文章，这些文章几乎都是从理论知识或者是“一种显然是这样”的角度来讲解C语言字符串常量，而且里面也有一些看似合理但是错误的言论。今天我打算在这里跟大家分享下我对于C语言字符串常量以及C语言字符串常量的存储的理解。首先我们来看看C语言标准[《C99》](http://port70.net/~nsz/c/c99/n1256.html#6.4.5)中对于字符串常量的定义
category: 程序员自我修养
comments: true
share: true
---

在我看来，程序员的个人修养的成长分为三个阶段。第一个阶段是懵懂阶段，在这个阶段不知道代码为什么要这么写，只知道这样写可以运行。第二个阶段是求知阶段，这个阶段是想知道为什么，然后搜索翻看资料，看看别人说这个是为什么。第三个阶段是探索，当不知道为什么的时候，运用自己的技术理论和事件去弄明白为什么。   

网络上也有很多关于字符串常量的讨论和文章，这些文章几乎都是从理论知识或者是“一种显然是这样”的角度来讲解C语言字符串常量，而且里面也有一些看似合理但是错误的言论。今天我打算在这里跟大家分享下我对于C语言字符串常量以及C语言字符串常量的存储的理解。

首先我们来看看C语言标准[《C99》](http://port70.net/~nsz/c/c99/n1256.html#6.4.5)中对于字符串常量的定义：

Description

A character string literal is a sequence of zero or more multibyte characters enclosed in double-quotes, as in "xyz". A wide string literal is the same, except prefixed by the letter L.

 The same considerations apply to each element of the sequence in a character string literal or a wide string literal as if it were in an integer character constant or a wide character constant, except that the single-quote ' is representable either by itself or by the escape sequence \', but the double-quote " shall be represented by the escape sequence \".

Semantics

 In translation phase 6, the multibyte character sequences specified by any sequence of adjacent character and wide string literal tokens are concatenated into a single multibyte character sequence. If any of the tokens are wide string literal tokens, the resulting multibyte character sequence is treated as a wide string literal; otherwise, it is treated as a character string literal.

 In translation phase 7, a byte or code of value zero is appended to each multibyte character sequence that results from a string literal or literals.66) The multibyte character sequence is then used to initialize an array of static storage duration and length just sufficient to contain the sequence. For character string literals, the array elements have type char, and are initialized with the individual bytes of the multibyte character sequence; for wide string literals, the array elements have type wchar_t, and are initialized with the sequence of wide characters corresponding to the multibyte character sequence, as defined by the mbstowcs function with an implementation-defined current locale. The value of a string literal containing a multibyte character or escape sequence not represented in the execution character set is implementation-defined.

 It is unspecified whether these arrays are distinct provided their elements have the appropriate values. If the program attempts to modify such an array, the behavior is undefined.

 EXAMPLE This pair of adjacent character string literals

          "\x12" "3"
produces a single character string literal containing the two characters whose values are '\x12' and '3', because escape sequences are converted into single members of the execution character set just prior to adjacent string literal concatenation.   

上面这段关于C语言字符串常量的表述，总结下，主要说了一下几点： 

1. 字符串常量使用双引号引起来的。
2. 字符串常量的类型是char类型的数组char []。
3. 字符串常量是以0(0在双引号中转义之后就是我们熟悉的\0)结尾。
4. 修改字符串常量的行为是未定义的。   
    

<br>
其中字符串常量是char []类型的，而不是const char []，原因很简单，因为K&R C中么有const，const是ANSI C增加的，请参考[《http://www.math.pku.edu.cn/teachers/qiuzy/c/ansi-kr-c.htm》](http://www.math.pku.edu.cn/teachers/qiuzy/c/ansi-kr-c.htm)。

对常量字符串的修改是未定义的，在操作系统和编译器实现的过程中，往往是会报没有权限的错误。对于这一点，我们可以看看下面的代码：
{% highlight C++ linenos %}
void print(char*)
{
    // ...
}

while (1)
{
    print("Hello");
}
{% endhighlight %}
如果在"Hello"在print内部被更改，我们传进去的是"Hello"，它在内存的内容已经不是"Hello"了，而是"I am fine, and you?"会不会很奇怪？对于这一点，不知道Ritchie在设计C语言的时候究竟是怎么想的。

那么C语言的字符串常量究竟是如何存储的呢？   

我们再来看一个Demo:
{% highlight C++ linenos %}
// main.c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

static char* sayHi = "Hi Fene!";
static char* sayHello = "Hello Fene!";
static char* dummyHi = "Hi Fene!";

void print()
{
    printf("sayHi:%s, sayHello:%s, dummyHi:%s\n", sayHi, sayHello, dummyHi);
}

int main()
{
   print();
   return 0;
}
{% endhighlight %}
编译命令如下：

	gcc -m32 -Wall main.c

我们使用objdump工具
	
	objdump -s a.out

dump出a.out的内容如下：
{% highlight objdump %}
0000043c <increase>:
Contents of section .rodata:
 80484f8 03000000 01000200 48692046 656e6521  ........Hi Fene!
 8048508 0048656c 6c6f2046 656e6521 00000000  .Hello Fene!....
 8048518 73617948 693a2573 2c207361 7948656c  sayHi:%s, sayHel
 8048528 6c6f3a25 732c2064 756d6d79 48693a25  lo:%s, dummyHi:%
 8048538 730a00                               s.. 
{% endhighlight %}
其中第一列是对应的内存单元地址，后面紧跟着的是几列是内存中内容的16进制数据。   

你会发现，刚刚Demo出现的所有字符串都被存放在.rodata区域，也就是只读区。每个字符串常量后面都跟了一个0。.rodata数据是在ELF文件被载入的时候，在内存中分配的内存，其生命周期同进程的生命周期。   


另外，还有一点，值得注意的是，sayHi和dummyHi所指向的内容都是"Hi Fene!"，但是你发发现在.rodata出现了2分"Hi Fene!"，也就意味着sayHi和dummyHi指针的值是不一样了，他们指向了不同的内存单元地址，我在Linux X86/X64/ARM上都有验证过。这也就是为什么不能通过判断字符串常量的指针来判断2个字符串是否相等的原因。但是在我看到的一些关于C语言字符串常量的资料中，有讲到，为了节省内存，代码中相同的字符串常量都指向同一个字符串常量，这一点我觉得有待于考证，至少Linux/X86/X64/ARM上不是这样的。