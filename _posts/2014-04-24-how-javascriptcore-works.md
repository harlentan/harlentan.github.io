---
layout: post
title: JavaScriptCore工作原理
tags: [JavaScriptCore]
keywords: JavaScriptCore编程, 深入浅出JavaScriptCore, JavaScriptCore工作原理
description: JavaScriptCore工作原理结合JavaScript和C++详细讲解了JavaScriptCore是如何工作的，JavaScript是如何调用C++的。
category: C/C++, JavaScriptCore
comments: true
share: true
---
##前言
JavaScript作为一种前端语言，特别是在WebApp概念炙热的当下，JavaScript变得越来越强大，一开始可能大多数JavaScript脚本只能被包含在HTML中在浏览器中运行，而现在JavaScript似乎什么都能做: JavaScript可以写HTTP Service(nodejs), 可以直接调用, 可以操作系统设备以及调用[WebAPI](https://developer.mozilla.org/en-US/Apps/Reference/General_Web_APIs), 在移动设备中甚至可以打电话、发短信、操作传感器等等。随着JavaScript引擎技术的不断发展和优化，JavaScript在某些场景下，毫不逊色与Java,C++。如此强大的JavaScript，了解其工作原理，有助于我们更好的去理解JavaScript，更好的去写JavaScript。本文主要针对JavaScriptCore来讲解其原理。讲解的过程中引用的一些avaScript内置对象，请参考[JavaScriptReference](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference)   

##JavaScript的解析   
我们从一个Demo开始
{% highlight JavaScript linenos %}
// Demo.js
var number = new Number(3.141592653589793);
number.hello = 'Hi';
console.log(number.toFixed(2));
{% endhighlight %}
JavaScriptCore首先会使用Lexer对这段代码进行解析，这里最明显的token有：new, Number, .操作符, hello, toFixed。可分为二类:

* 基础操作符：new, .操作符

* 属性：hello, toFixed   


JavaScriptCore为实现基础操作符，实现了一系列的OPCODE，主要定义在JavaScriptCore/bytecode/Opcode.h中，本文中讲到的OPCODE如下：
{% highlight C++ linenos %}
// Opcode.h
    macro(op_new_object, 4) \
    macro(op_put_by_id, 9) \
    macro(op_get_by_pname, 7) \
{% endhighlight %}
其中`op_new_object`就是Demo.js中的new。`op_put_by_id`就是Demo.js中的给number对象添加hello属性，`op_get_by_pname`就是Demo.js中的number.toFixed获取toFixed属性。

##JavaScript对象的创建    
通过上面的解析，已经解析出了new的OPCODE: `op_new_object`。JavaScriptCore中最重要的对象就是JSGlobalObject。所有内置对象的Constructor的在JSGlobalObject对象初始化的过程中被Binding到JSGlobalObject上。这个Binding的过程其实就是<Key, Value>的一个Property Map。在JSObject的初始化过程中，先创建NumberConstructor并与`Number`进行映射，这个Constructor就是后面用来new Number的。那么在JavaScript中`var number = new Number(3.141592653589793)`语句做的事情就是，在JSGlobalObject的Property Map表中查找`Number`这string，如果找到了，而且OPCODE是`op_new_object`, JavaScriptCore就会通过`Number`对应的Value `NumberConstructor`来创建Number对象：
{% highlight C++ linenos %}
// NumberConstructor.cpp
static EncodedJSValue JSC_HOST_CALL constructWithNumberConstructor(ExecState* exec)
{
    NumberObject* object = NumberObject::create(exec->vm(), asInternalFunction(exec->callee())->globalObject()->numberObjectStructure());
    double n = exec->argumentCount() ? exec->argument(0).toNumber(exec) : 0;
    object->setInternalValue(exec->vm(), jsNumber(n));
    return JSValue::encode(object);
}
{% endhighlight %}
create了一个`NumberObject`，最后将`NumberObject`的指针通过`encode`之后返回。对于NumberObject::create的逻辑，这里先不深入。上面一段代码，最终要的语句是：`return JSValue::encode(object);`。难道是将object指针传到JavaScript中去？但是JavaScript中没有指针的概念啊。我们先来看看这个encode的实现：
{% highlight C++ linenos %}
// JSCJSValueInlines.h
inline JSValue::JSValue(const JSCell* ptr)
{
    u.asInt64 = reinterpret_cast<uintptr_t>(const_cast<JSCell*>(ptr));
}

inline EncodedJSValue JSValue::encode(JSValue value)
{
    return value.u.asInt64;
}
{% endhighlight %}
这段代码非常容易理解，首先NumberObject指针传进去会被隐式转换成JSValue，转换成JSValue的过程仅仅只是将还真转换成一个64位的int型。而所谓的encode，就是直接将这个Int64直接返回到JavaScript中去。而这个Int64在C++中实际又是一个指针，所以当这个Int64从JavaScript中传回来的时候，又可以通过reinterpre_cast转化成一个C++指针在C++中做相应的操作。讲到这里，我们基本 可以认为JavaScript中的对象就是Int64的整数，代表C++中的对象指针。    

##属性访问   
上面我们已经分析出JavaScript对象的本质了，再来看看属性访问，就比较简单了。上面已经提到过，JavaScript中对象的属性在C++中就是<K, Value>的形式存在的。当执行`number.hello = 'Hi';`的时候，解析出op\_put\_by\_id，此时就会将number转换成C++中的NumberObject的对象指针，然后在NumberObject的属性表里面添加hello属性，其值是`Hi`。同样，`number.toFixed(2)`也是去C++中找到toFixed，然后调用。


##总结    
上面的分析你可以发现，JavaScript对象跟C++中的对象指针是唯一对应的。也就是说只要C++能做到的，JavaScript也能做到。