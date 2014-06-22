---
layout: post
title: pthread_cond_signal虚假唤醒(spurious wakeup)
tags: [C/C++, Linux]
keywords: 条件变量, pthread_cond_signal虚假唤醒(sprious wakeup), pthread_cond_wait遗漏消息, Linux条件变量
description: 最近在使用Linux条件变量的时候，经过反复测试发现，`pthread_cond_signal`有时候会唤起多个正在`pthread_cond_wait`的线程。后来通过查阅[IEEE Std 1003.1, 2004](http://pubs.opengroup.org/onlinepubs/009695399/toc.htm)中关于pthread\_cond\_signal虚假唤醒(spurious wakeup)的解释如下：
comments: true
share: true
---

##虚假唤醒

最近在使用Linux条件变量的时候，经过反复测试发现，`pthread_cond_signal`有时候会唤起多个正在`pthread_cond_wait`的线程。后来通过查阅[IEEE Std 1003.1, 2004](http://pubs.opengroup.org/onlinepubs/009695399/toc.htm)中关于pthread\_cond\_signal虚假唤醒(spurious wakeup)的解释如下：

>On a multi-processor, it may be impossible for an implementation of pthread_cond_signal() to avoid the unblocking of more than one thread blocked on a condition variable.

根据这个解释，在多处理器系统上，pthread\_cond\_signal是很有可能唤醒多个pthread\_cond\_wait()的线程。也就意味着当一个线程中，pthread\_cond\_wait()返回的时候，不一定代表条件已经满足了，需要在程序中做额外的判断来检测是否真的已经满足条件了：


{% highlight C++ linenos %}
pthread_mutex_lock(&lock);
while (condition_is_false) {
    pthread_cond_wait(&cond, &lock);
}
pthread_mutex_unlock(&lock);
{% endhighlight %}

事实上，[IEEE Std 1003.1, 2004](http://pubs.opengroup.org/onlinepubs/009695399/toc.htm)中有提到，虚假唤醒(spurious wakeup)是被允许的，而且鼓励程序开发者在pthread\_cond\_wait()返回的时候对条件进行重新检查，只有在条件满足的情况下才继续往下执行，否则就需要继续等待了。

关于多处理器系统出现虚假唤醒(sprious wakeup)的原因，我的理解是因为多处理器上，多线程共享的数据需要在多核处理器上cache进行更新和拷贝的原因。关于多核多线程请参考[《利用多核多线程进行程序优化》](http://www.ibm.com/developerworks/cn/linux/l-cn-optimization/)

##消息遗漏

对于pthread\_cond\_signal或者pthread\_cond\_broadcast来说，除了需要在pthread\_cond\_wait()返回的时候，重新对条件进行检查和评估以外，还有一件事情就是需要解决消息遗漏的问题。   


根据[pthread\_cond_wait](http://linux.die.net/man/3/pthread_cond_wait)的定义，需要在pthread\_cond\_wait调用前后必须进行加锁和解锁操作。原因是因为如果在一个线程调用pthread\_cond\_wait的过程中但未进入block状态，此时有线程调用了pthread\_cond\_signal或者pthread\_cond\_broadcast，那么此次消息将被遗漏掉，因为没有任何线程在pthread\_cond\_wait的block状态。在pthread\_cond\_wait的实现内部，首先会解锁，然后进入block状态，解锁和进入block必须合并成一个原子操作，这样就保证了在pthread\_cond\_wait之后调用的pthread\_cond\_signal不会被以后掉。


但是对于多线程来说，pthread\_cond\_wait不能保证一定在pthread\_cond\_signal之后执行，也就意味着，当pthread\_cond\_wait进入block之后，已经错过了pthread\_cond\_signal。因为已经错过了pthread\_cond\_signal，很有可能会导致该线程永远block下去。通常这类问题的解决办法是设置一个pthread\_cond\_signal或者pthread\_cond\_broadcast的计数器count，在调用pthread\_cond\_wait之前先对这个count进行判断，如果`count != 0`
则说明已经错过了消息，可以不用等待，直接往下执行即可：

{% highlight C++ linenos %}
if (!count) {
	pthread_mutex_lock(&lock);
	while (condition_is_false) {
    	pthread_cond_wait(&cond, &lock);
	}
	pthread_mutex_unlock(&lock);
}
{% endhighlight %}