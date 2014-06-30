---
layout: post
title: node-webkit的实现原理详解
tags: [WebKit]
keywords: Node-WebKit, Nodejs, WebKit, Nodejs WebKit
description: Node-WebKit最终呈现给开发者的就是在HTML的JS中可以调用Nodejs的所有API。HTML的JS引擎是Google V8，Nodejs的JS引擎也是Google V8。WebKit和Nodejs都是使用Google V8提供binding。为了让运行在WebKit上的HTML中的JS可以调用Nodejs的所有API，只需要将WebKit和Nodejs的Google V8打通。为了打通Node-WebKit，除了要打通JS引擎以外，还需要打通消息事件循环Loop。
comments: true
share: true
---

##Node like

随着Nodejs的火爆，越来越多的node like以及跟Nodejs结合的框架出现。Node-WebKit就是其中一个。Nodejs之所以越来越受欢迎是因为Nodejs让JS有了很多普通JS不具备的能力，例如network，filesystem，等等。

Node-WebKit之所以能够被广大前端开发者所认可也是因为Node-WebKit能做WebKit做不了的事情，我觉得Node-WebKit的真正意义在于它使得运行在WebKit上的WebApp或者网页具备了Nodejs的所有能力，Node-WebKit其实是突破了WebKit的很多限制。

##Node-WebKit基本原理

<center>![Alt Text](/images/node-webkit.svg)</center>

Node-WebKit最终呈现给开发者的就是在HTML的JS中可以调用Nodejs的所有API。HTML的JS引擎是Google V8，Nodejs的JS引擎也是Google V8。WebKit和Nodejs都是使用Google V8提供binding。


为了让运行在WebKit上的HTML中的JS可以调用Nodejs的所有API，只需要将WebKit和Nodejs的Google V8打通。   

在打通JS引擎之后，还有一件重要的事情需要做：消息循环的融合。因为WebKit和Nodejs都有自己的Looper提供消息循环处理机制，WebKit在Android或者Chromium上，使用的是Java的Loop，而Nodejs则是使用libuv的uv\_loop\_t，也就是说他俩属于不同的世界，在HTML中调用Nodejs的API的时候，WebKit和Nodejs消息事件循环都无法感知到对方的存在。

为了打通Node-WebKit，除了要打通JS引擎以外，还需要打通消息事件循环Loop。


##V8基础知识

在讲解Node-WebKit是如何打通Nodejs和WebKit的JS引擎和JS Bindings的时候，有必要现对一些V8的基本概念进行回顾，以便于我们更好的理解。这里只是讲讲本文会用到一些Google V8的技术。更多知识请参考我在CSDN的专栏[《Google V8编程详解》](http://blog.csdn.net/column/details/googlev8.html)。

###Context

Google设计Context的概念是为了要解决WebKit的一个问题：不同的window和iframe中都是一个全新的JavaScript执行环境。

因为JavaScript的一些特性的原因，使得任何JavaScript代码都可以修改全局的一些对象或者属性，如果所有的iframe或者window里面都是同一个JavaScript执行环境，这使得JavaScript运行环境容易遭到污染。

<center>![Alt Text](/images/v8context.svg)</center>

V8的Context是分层的。任何一个对象都必须执行在特定的Context。这也就是`Context::Scope context_scope(context);`的作用。这句话就是为了告诉V8，接下来JavaScript将要执行在哪个Context中。此外，Context还有一个方法`Context::Enter()`，也可以用来告诉V8，后面的JavaScript将要在调用`Enter()`的Context中执行。  

其实`Context::Scope context_scope(context);`这句话内部也仅仅只是调用`context->Enter();`而已。

上面这张图，先将Context A作为JavaScript的执行环境，紧接着又将Context B作为当前JavaScript的执行环境，最后又将Context C作为当前JavaScript的执行环境。    
其对应的C++代码， 就是执行了一下操作：

{% highlight C++ linenos %}
// ...
Handle<Context> contextA = Context::New();
Handle<Context> contextB = Context::New();
Handle<Context> contextC = Context::New();

// do something.

contextA->Enter();

// do something or execute some javascript

contextB->Enter();

// do something or execute some javascript

contextC->Enter();

{% endhighlight %}
每当Enter()进入到一个新的Context之后，当前JavaScript的执行环境都是一个新的，这里所谓的新的执行环境包括新的JavaScript global，不会受到上一个Context的干扰和污染。

上面是正向的，逆向的情况这是`Context::Exit()`。当`contextC->Exit()`，当前JavaScript的执行环境将切换到上一个Context，在这里指的就是ContextB。


###Context交叉访问

上面讲到讲到不同的Context其实是一个互不干扰的全新的JavaScript执行环境，包括JavaScript global对象。Google V8对Context的这种控制机制是源于[same origin policy](http://en.wikipedia.org/wiki/Same-origin_policy)，这个策略最早是在[Netscape Navigator 2](http://en.wikipedia.org/wiki/Netscape_Navigator_2)被提出的。这个安全策略规定了{protocol, host, port}作为唯一的origin的标示，不同的origin不能互相访问。

在V8中，这个origin指的就是Context。默认情况下，不同的Context是无法互相访问的，也即是ContextA中的JavaScript对象是无法访问ContextB中的JavaScript对象的。如果需要访问非当前的Context中的JavaScript对象，必须要求这2个Context的security token相同。V8提供了`Context:SetSecurityToken()`和`Context::SetSecurityToken()`用来操作Context的security token，这个token就是Contxt之间约定的一个字符串而已。


##Nodejs对象安装
为了尽量减小对Chromium以及Nodejs的改动，以及降低Node和Chromium的耦合性，Node-WebKit将WebKit的bindings和Nodejs的bindings没有放在同一个Context中。

###加载Nodejs
Nodejs会New一个Context，然后设置一个security toke, 并在这个Context中初始化Nodejs的一些JavaScript对象。

{% highlight C++ linenos %}
      // ...

      char* argv[] = { const_cast<char*>("node"), NULL, NULL };
      v8::Isolate* isolate = v8::Isolate::GetCurrent();
      v8::HandleScope scope(isolate);

      v8::RegisterExtension(new nwapi::WindowBindings());
      const char* names[] = { "window_bindings.js" };
      v8::ExtensionConfiguration extension_configuration(1, names);


      v8::Local<v8::Context> context = v8::Context::New(isolate
      	,&extension_configuration);
      node::g_context.Reset(isolate, context);
      context->SetSecurityToken(v8_str("nw-token"));
      context->Enter();
      context->SetEmbedderData(0, v8_str("node"));

      node::SetupContext(argc, argv, context);
{% endhighlight %}
可能不同版本的不一样，但是大致代码都差不多，在WebKit初始化自己的Context之前，先将Nodejs的Context New出来保存在node::g\_context中，并设置自己的securityToken，最后`node::SetupContext`就是将Nodejs加载到WebKit进程当中。    
此时，Nodejs所有的API以及一些全局对象都加载到了`node::g_context`当中。


###初始化WebKit Context

Nodejs的加载要早于WebKit的JavaScript执行环境的初始化。[《WebKit/WebCore中JavaScript在JavaScriptCore上的执行》](http://www.fenesky.com/blog/2014/04/11/WebKit-script-execution-jsc.html)里面也有讲过，无论是JavaScriptCore还是V8，WebKit的所有JavaScript都是在ScriptController中执行的。WebKit的JavaScript的执行环境也是在ScriptController初始化的过程中构建起来的。   

在ScriptController初始化完毕之后，会调用didCreateScriptContext来告知JavaScript执行环境已经初始化完毕：
{% highlight C++ linenos %}
bool V8WindowShell::initialize()
{
    TRACE_EVENT0("v8", "V8WindowShell::initialize");
    TRACE_EVENT_SCOPED_SAMPLING_STATE("Blink", "InitializeWindow");

    v8::HandleScope handleScope(m_isolate);

    createContext();

    if (!m_perContextData)
        return false;

    v8::Handle<v8::Context> context = m_perContextData->context();
    v8::Context::Scope contextScope(context);


    // ...

    m_frame->loader().client()->didCreateScriptContext(context
    	,m_world->extensionGroup()
    	, m_world->worldId());

 }
{% endhighlight %}
这个执行过程中，最重要的是WebKit New了一个Context，并Enter了。也就是说此时JavaScript执行环境从Nodejs的node::g\_context切换到了WebKit的JavaScript执行环境。


###Context互通

根据上面的讲解，我们已经知道，到现在已经有了2个Context，一个是Nodejs的node::g\_context，一个是WebKit的Context，由于Nodejs的Context早于WebKit的Context，此时整个WebKit的JavaScript的执行环境切换到了WebKit的Context。

也就意味这所有的HTML的JavaScript都会执行在WebKit的Context中，而不会是Nodejs的node::g\_context。为了使得WebKit的Context中能够使用所有Nodejs的API，就需要WebKit的Context能够访问node::g\_context中的对象。

根据之前的V8基础理论知识的讲解，WebKit的Context必须跟Nodejs的node::g\_context的SecurityToken保持一致才可以互相访问。

{% highlight C++ linenos %}
bool ShellContentRendererClient::WillSetSecurityToken(
    blink::WebFrame* frame,
    v8::Handle<v8::Context> context) {
  GURL url(frame->document().url());
  VLOG(1) << "WillSetSecurityToken: " << url;
  if (goodForNode(frame)) {
    VLOG(1) << "GOOD FOR NODE";
    // Override context's security token
    v8::Isolate* isolate = v8::Isolate::GetCurrent();
    v8::HandleScope scope(isolate);
    v8::Local<v8::Context> g_context =
      v8::Local<v8::Context>::New(isolate, node::g_context);
    context->SetSecurityToken(g_context->GetSecurityToken());
    frame->document().securityOrigin().grantUniversalAccess();

    int ret = 0;
    RenderViewImpl* rv = RenderViewImpl::FromWebView(frame->view());
    rv->Send(new ViewHostMsg_GrantUniversalPermissions(rv->GetRoutingID(), &ret));

    return true;
  }

  // ...
}
{% endhighlight %}
WillSetSecurityToken的参数context就是WebKit的Context。`context->SetSecurityToken(g_context->GetSecurityToken());`将WebKit的SecurityToken设置为Nodejs中node::g\_context的SecurityToken。这样一来在WebKit的Context中就可以访问Nodejs的node::g\_context的JavaScript对象了。

###加载Nodejs API

{% highlight C++ linenos %}
void ShellContentRendererClient::InstallNodeSymbols(
    blink::WebFrame* frame,
    v8::Handle<v8::Context> context,
    const GURL& url) {
    v8::Isolate* isolate = v8::Isolate::GetCurrent();
    v8::HandleScope scope(isolate);
    v8::Local<v8::Context> g_context =
    v8::Local<v8::Context>::New(isolate, node::g_context);

    static bool installed_once = false;

    v8::Local<v8::Object> nodeGlobal = g_context->Global();
    v8::Local<v8::Object> v8Global = context->Global();

    // Use WebKit's console globally
    nodeGlobal->Set(v8::String::NewFromUtf8(isolate, "console"),
                  v8Global->Get(v8::String::NewFromUtf8(isolate, "console")));

    // Do we integrate node?
    bool use_node = goodForNode(frame);

    // Test if protocol is 'nw:'
    // test for 'about:blank' is also here becuase window.open would
    // open 'about:blank' first // FIXME
    bool is_nw_protocol = url.SchemeIs("nw") || !url.is_valid();

    if (use_node || is_nw_protocol) {
      frame->setNodeJS(true);

      v8::Local<v8::Array> symbols = v8::Array::New(isolate, 4);
      symbols->Set(0, v8::String::NewFromUtf8(isolate, "global"));
      symbols->Set(1, v8::String::NewFromUtf8(isolate, "process"));
      symbols->Set(2, v8::String::NewFromUtf8(isolate, "Buffer"));
      symbols->Set(3, v8::String::NewFromUtf8(isolate, "root"));

      g_context->Enter();
      for (unsigned i = 0; i < symbols->Length(); ++i) {
          v8::Local<v8::Value> key = symbols->Get(i);
          v8::Local<v8::Value> val = nodeGlobal->Get(key);
          v8Global->Set(key, val);
      }
      g_context->Exit();

      // ...
    }

 }
{% endhighlight %}
有了之前V8的基础知识，再来看这段代码就会轻松的多。有一点需要明确的是，此时执行JavaScript所在的Context是WebKit的Context。

这段代码的主要逻辑是将Nodejs在node::g\_context中初始化的对象取出来set到WebKit中的Context中去。就是这么简单，由于Nodejs的node::g\_context和WebKit的Context的SecurityToken是相同的，那么可以直接取出Nodejs node::g
\_context上的`nodeGlobal`，并将挂载在nodeGlobal上的global, process, Buffer这些Nodejs的global对象取出来，挂载到WebKit Context的global对象`v8Global`上去。这样，其实就完成了Nodejs所有API的挂载和对接。

##消息事件循环融合

Chromium在[https://github.com/chromium/chromium/tree/trunk/base](https://github.com/chromium/chromium/tree/trunk/base)下实现了自己的Looper。主要用来对Chromium的事件分发处理。但是Nodejs使用的libuv来提供的[uv_loop](http://nikhilm.github.io/uvbook/basics.html#event-loops)。为了是的Chromium和Nodejs的事件能够融合，Node-WebKit使用libuv实现了Chromium的[MessagePumpUV ](https://github.com/zcbenz/cefode-chromium/blob/master/base/message_pump_uv.cc)。将所有的事件处理交给libuv的uv\_loop\_t来处理。

<center>![Alt Text](/images/nw-message_loop.svg)</center>

如图所示，正常情况，在没有事件的情况下，message loop都处于Wait状态并阻塞，当Nodejs或者Chromium有事件的时候，会将uv\_loop\_t激发，message loop被唤醒。   

message loop阻塞Wait的逻辑如下：
{% highlight C++ linenos %}
    if (delayed_work_time_.is_null()) {
      uv_run(loop, UV_RUN_ONCE);
    } else {
      TimeDelta delay = delayed_work_time_ - TimeTicks::Now();
      if (delay > TimeDelta()) {
        uv_timer_start(&delay_timer, timer_callback,
                       delay.InMilliseconds(), 0); 
        uv_run(loop, UV_RUN_ONCE);
        uv_idle_stop(&idle_handle);
        uv_timer_stop(&delay_timer);
      } else {
        // It looks like delayed_work_time_ indicates a time in the past, so we
        // need to call DoDelayedWork now.
        delayed_work_time_ = TimeTicks();
      }   
    }
{% endhighlight %}
当没有事件的时候，`uv_run(loop, UV_RUN_ONCE)`会一直阻塞，等待事件的到来。这种方式处理Nodejs事件没问题。Chromium的事件是如何被处理的呢？

我们来看看ScheduleWork的实现：
{% highlight C++ linenos %}
void MessagePumpUV::ScheduleWork() {
  // Since this can be called on any thread, we need to ensure that our Run
  // loop wakes up.
  uv_async_send(wakeup_event_ref_);
}
{% endhighlight %}
当Chromium有事件需要处理的时候，会调用ScheduleWork。MessagePumpUV重载了ScheduleWork方法，ScheduleWork直接使用uv\_async\_send来激活message loop，来解除`uv_run(loop, UV_RUN_ONCE)`的阻塞，然后在doWork中处理Chromium的事件。


##结束语

到此为止，Node-WebKit的核心，都已经分析完毕。我之前在Android WebKit和Android Chromium上也做过Node-WebKit，基本思想都是这个。现将Nodejs API在WebKit中加载进来，然后将Looper进行融合。