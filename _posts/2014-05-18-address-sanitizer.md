---
layout: post
title: AddressSanitizier(ASAN)
tags: [C++]
keywords: AddressSanitizier, ASAN, AndressSanitizier on Android, ASAN on Android
description: AddressSanitizier(ASAN)Android上的使用方法
comments: true
---

&emsp;&emsp;[Address-Sanitizier(ASAN)](https://code.google.com/p/address-sanitizer/wiki/AddressSanitizer)是Google开发的一款用于检查C++内存错误的工具，类似于Valgrind。与Valgrind不同的是，使用ASAN检查内存，需要对应用程序进行重新编译。在编译阶段，需要将一些内存操作函数替换成ASAN定义和实现的函数，例如malloc, free, memcpy等等， LLVM 3.1和GCC 4.8以后，已经集成了ASAN。在其他平台上只需要添加编译选项-fsanitize=address然后优化选项在-O1以及以上即可。目前越来越多的程序都已经开始支持ASAN了，例如chromium, firefox等等。

&emsp;&emsp;在Android 4.2以后已经集成了ASAN。源码位于external/compiler-rt/lib/asan。只有在eng版本中，ASAN才会被编译出来。如果要使用ASAN测试Android的动态库或者可执行程序，需要在Android.mk中添加`LOCAL_ADDRESS_SANITIZER := true`, 编译过程中有可能会有很多报错，但是这些报错都跟头文件包含相关，ASAN会对头文件包含造成影响，稍微修改头文件包含以及定义即可。ASAN会编译出libasan_preload.so和asanwrapper。有2中方式用来运行测试Android应用的内存。一种就是使用`asanwrapper  /system/bin/xxx  --options`。但是不是所有的Android程序都可以使用asanwrapper来启动。但是从asanwrapper.cc

{% highlight C++ linenos %}
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string>

void usage(const char* me) {
  static const char* usage_s = "Usage:\n"
    "  %s /system/bin/app_process <args>\n"
    "or, better:\n"
    "  setprop wrap.<nicename> %s\n";
  fprintf(stderr, usage_s, me, me);
  exit(1);
}

void env_prepend(const char* name, const char* value, const char* delim) {
  const char* value_old = getenv(name);
  std::string value_new = value;
  if (value_old) {
    value_new += delim;
    value_new += value_old;
  }
  setenv(name, value_new.c_str(), 1);
}

int main(int argc, char** argv) {
  if (argc < 2) {
    usage(argv[0]);
  }
  char** args = new char*[argc];
  // If we are wrapping app_process, replace it with app_process_asan.
  // TODO(eugenis): rewrite to <dirname>/asan/<basename>, if exists?
  if (strcmp(argv[1], "/system/bin/app_process") == 0) {
    args[0] = (char*)"/system/bin/app_process_asan";
  } else {
    args[0] = argv[1];
  }

  for (int i = 1; i < argc - 1; ++i)
    args[i] = argv[i + 1];
  args[argc - 1] = 0;

  env_prepend("ASAN_OPTIONS", "debug=1,verbosity=1", ",");
  env_prepend("LD_LIBRARY_PATH", "/system/lib/asan", ":");
  env_prepend("LD_PRELOAD", "/system/lib/libasan_preload.so", ":");

  printf("ASAN_OPTIONS: %s\n", getenv("ASAN_OPTIONS"));
  printf("LD_LIBRARY_PATH: %s\n", getenv("LD_LIBRARY_PATH"));
  printf("LD_PRELOAD: %s\n", getenv("LD_PRELOAD"));

  execv(args[0], args);
}
{% endhighlight %}
通过这段源码，你可以发现，asanwrapper.cc制作了一件事情，就是让libasan_preload.so程序运行的任何被执行之前，被load到进程中。所以，使用ASAN测试Android APK的应用，只需要在app\_process Android.mk中添加ASAN编译选项`LOCAL_ADDRESS_SANITIZER := true`，然后使用[asan\_device\_setup.sh](https://llvm.org/viewvc/llvm-project/compiler-rt/trunk/lib/asan/scripts/asan_device_setup.sh?diff_format=f&pathrev=200199&logsort=date&sortby=rev&view=markup&revision=200199)来启动设备。这里也有一份官方的[ASAN android](https://code.google.com/p/address-sanitizer/wiki/Android)使用和配置文档，但是该使用方法比较陈旧。