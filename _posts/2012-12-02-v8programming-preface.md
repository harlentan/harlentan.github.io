---
layout: post
title: Google V8编程详解（序）
tags: [blog]
keywords: V8编程, Google V8, V8编程详解, Google V8编程详解, CloudApp, HTML5
description: V8编程详解
comments: true
---

  应用程序发展到今天，应用程序的概念也在不断地发生着变化，WiKi的解释是这样的：“应用程序指为完成某项或多项特定工作的计算机程序”。这里所指的应用程序在软件行的今天，绝大多数指的是需要经过下载安装在本定机器上运行的程序，称之为本地应用。

<!--more-->
  而目前国内很多IT公司都在部署自己的移动互联网战略，主推CloudApp云应用，如阿里云OS、百度云应用。CloudApp正在形成一种新的应用程序形式，即不需要安装即可使用的程序。体现了CloudApp热部署的特点，这也是JS的特性。这样就意味着当前的CloudApp使用js来编写，当然，也少不了HTML5。JS作为一种热部署性高，灵活性强的语言，自然成了CloudApp的首选。目前主流的JS引擎有JavaScript engine和Google V8(简称V8)。   

  关于这两个JS引擎业界也有不少争论，这里我只讲关于V8的部分。V8作为JS的解释器，能将JS世界和C/C++世界打通，JS可以直接调用C/C++接口，使得Cloud App能够具有和本地应用进行交互的可能。虽然这一点不是V8独有的，Qt的QML（类似于JS）早就已经具备直接构造和调用C++对象的能力。JS世界和C/C++世界的融合，使得Cloud App的扩张有了无限的可能。
