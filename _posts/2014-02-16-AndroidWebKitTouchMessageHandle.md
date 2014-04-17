---
layout: post
title: Android WebKit消息处理(二)Touch事件的分发处理
tags: [blog]
keywords: Android WebKit消息处理(二)Touch事件的分发处理, fenesky, android, WebKit，谭海燕
description: Android WebKit消息处理(二)Touch事件的分发处理, 详细讲解了Android WebKit Touch事件的传递和处理
comments: true
---
上一章[《Android WebKit消息处理》](http://www.fenesky.com/blog/2014/02/11/Android-WebKit-MsgHandle.html)讲到了Android WebKit主体初始化流程以及消息处理框架的搭建。这一章主要讲讲Android WebKit touch事件的处理与分发。

<!--more-->

##WebViewCore初始化过程



{% highlight java linenos %}
/* Initialize private data within the WebCore thread. 
 */  
private void initialize() {  
    /* Initialize our private BrowserFrame class to handle all 
     * frame-related functions. We need to create a new view which 
     * in turn creates a C level FrameView and attaches it to the frame. 
     */  
    mBrowserFrame = new BrowserFrame(mContext, this, mCallbackProxy,  
            mSettings, mJavascriptInterfaces);  
    mJavascriptInterfaces = null;  
    // Sync the native settings and also create the WebCore thread handler.  
    mSettings.syncSettingsAndCreateHandler(mBrowserFrame);  
    // Create the handler and transfer messages for the IconDatabase  
    WebIconDatabaseClassic.getInstance().createHandler();  
    // Create the handler for WebStorageClassic  
    WebStorageClassic.getInstance().createHandler();  
    // Create the handler for GeolocationPermissions.  
    GeolocationPermissionsClassic.getInstance().createHandler();  
  
    // ...  
    // ...     
  
    // The transferMessages call will transfer all pending messages to the  
    // WebCore thread handler.  
    mEventHub.transferMessages();  
  
    // Send a message back to WebView to tell it that we have set up the  
    // WebCore thread.  
    if (mWebViewClassic != null) {  
        Message.obtain(mWebViewClassic.mPrivateHandler,  
                WebViewClassic.WEBCORE_INITIALIZED_MSG_ID,  
                mNativeClass, 0).sendToTarget();  
    }
}         
{% endhighlight %}
在WebViewCore初始化结束之后，会给WebViewClassic发送WEBCORE\_INITIALIZED\_MSG_ID消息。   
{% highlight java linenos %}
case WEBCORE_INITIALIZED_MSG_ID:  
    // nativeCreate sets mNativeClass to a non-zero value  
    String drawableDir = BrowserFrame.getRawResFilename(  
            BrowserFrame.DRAWABLEDIR, mContext);  
    nativeCreate(msg.arg1, drawableDir, ActivityManager.isHighEndGfx());  
    if (mDelaySetPicture != null) {  
        setNewPicture(mDelaySetPicture, true);  
        mDelaySetPicture = null;  
    }      
    if (mIsPaused) {  
        nativeSetPauseDrawing(mNativeClass, true);  
    }      
    mInputDispatcher = new WebViewInputDispatcher(this,  
            mWebViewCore.getInputDispatcherCallbacks());  
    break;  
{% endhighlight %}
在WebViewClassic收到来自WebViewCore的初始化完毕的消息之后，会创建WebViewInputDispatcher对象：   
{% highlight java linenos %}
mInputDispatcher = new WebViewInputDispatcher(this,  
        mWebViewCore.getInputDispatcherCallbacks()); 
{% endhighlight %}
WebViewClassic的mInputDispatcher对象是touch事件分发处理的核心。下面讲的Touch事件的分发处理绝大部分是在WebViewInputDispatcher中进行的。  
WebViewInputDispatcher构造函数的原型如下：
{% highlight java linenos %}
public WebViewInputDispatcher(UiCallbacks uiCallbacks, WebKitCallbacks webKitCallbacks)  
{% endhighlight %}
+ uiCallbacks: WebViewClassic的mPrivateHandler   
+ webKitCallbacks: WebViewCore的mEventHub   

WebViewInputDispatcher把touch输入事件分为Ui事件和WebKit事件（就是上面WebCore事件），更具WebViewClassic在创建WebViewInputDispatcher的时候传入的参数，会分别创建自己的mUiHandler和mWebKitHandler。同时保存mUiCallbacks和mWebKitCallbacks以便于分别向WebViewClassic和WebViewCore发送消息：
{% highlight java linenos %}
public WebViewInputDispatcher(UiCallbacks uiCallbacks, WebKitCallbacks webKitCallbacks) {  
    this.mUiCallbacks = uiCallbacks;  
    mUiHandler = new UiHandler(uiCallbacks.getUiLooper());  
  
    this.mWebKitCallbacks = webKitCallbacks;  
    mWebKitHandler = new WebKitHandler(webKitCallbacks.getWebKitLooper());  
  
    ViewConfiguration config = ViewConfiguration.get(mUiCallbacks.getContext());  
    mDoubleTapSlopSquared = config.getScaledDoubleTapSlop();  
    mDoubleTapSlopSquared = (mDoubleTapSlopSquared * mDoubleTapSlopSquared);  
    mTouchSlopSquared = config.getScaledTouchSlop();  
    mTouchSlopSquared = (mTouchSlopSquared * mTouchSlopSquared);  
}
{% endhighlight %}   
其中mUiHandler是依附在UI线程上，mWebKitHandler是依附在WebCore线程上。
由于WebView是继承于View，所以当点击页面的时候，WebView的onTouchEvent会被调用，从而触发WebViewClassic的onTouchEvent：  
{% highlight java linenos %}
@Override  
public boolean onTouchEvent(MotionEvent ev) {  
    if (mNativeClass == 0 || (!mWebView.isClickable() && !mWebView.isLongClickable())) {  
        return false;  
    }      
  
    if (mInputDispatcher == null) {  
        return false;  
    }      
  
    if (mWebView.isFocusable() && mWebView.isFocusableInTouchMode()  
            && !mWebView.isFocused()) {  
        mWebView.requestFocus();  
    }      
  
    if (mInputDispatcher.postPointerEvent(ev, getScrollX(),  
            getScrollY() - getTitleHeight(), mZoomManager.getInvScale())) {  
        mInputDispatcher.dispatchUiEvents();  
        return true;  
    } else {  
        Log.w(LOGTAG, "mInputDispatcher rejected the event!");  
        return false;  
    }      
}    
{% endhighlight %} 
这是Android WebKit的所有touch输入事件的入口。touch事件通过postPointerEvent进入，然后对UI事件和WebKit事件进行分拣，派遣到对应的消息队列。  
##Ui事件和WebKit事件的分拣
在消息分拣过程中，会判断touch输入事件是否真的需要派发给WebKit.
{% highlight java linenos %}
private void enqueueEventLocked(DispatchEvent d) {  
     if (!shouldSkipWebKit(d)) {  
         enqueueWebKitEventLocked(d);  
     } else {  
         enqueueUiEventLocked(d);  
     }  
 }  
  
 private boolean shouldSkipWebKit(DispatchEvent d) {  
     switch (d.mEventType) {  
         case EVENT_TYPE_CLICK:  
         case EVENT_TYPE_HOVER:  
         case EVENT_TYPE_SCROLL:  
         case EVENT_TYPE_HIT_TEST:  
             return false;  
         case EVENT_TYPE_TOUCH:  
             // TODO: This should be cleaned up. We now have WebViewInputDispatcher  
             // and WebViewClassic both checking for slop and doing their own  
             // thing - they should be consolidated. And by consolidated, I mean  
             // WebViewClassic's version should just be deleted.  
             // The reason this is done is because webpages seem to expect  
             // that they only get an ontouchmove if the slop has been exceeded.  
             if (mIsTapCandidate && d.mEvent != null  
                     && d.mEvent.getActionMasked() == MotionEvent.ACTION_MOVE) {  
                 return true;  
             }  
             return !mPostSendTouchEventsToWebKit  
                     || mPostDoNotSendTouchEventsToWebKitUntilNextGesture;  
     }  
     return true;  
 } 
 {% endhighlight %}
通常情况下，几乎所有的touch事件都需要派发给WebKit，除非WebKit不需要：
在Document初始化过程中。
在FrameLoader stopLoad之后。
Document被析构。
这两种情况都会导致Document::removeAllEventListerners()被调用：  
通常情况下，几乎所有的touch事件都需要派发给WebKit，除非WebKit不需要：  
1. 在Document初始化过程中。  
2. 在FrameLoader stopLoad之后。  
3. Document被析构。  
这两种情况都会导致Document::removeAllEventListerners()被调用：
{% highlight java linenos %}
void Document::removeAllEventListeners()  
{  
#if ENABLE(TOUCH_EVENTS)  
    Page* ownerPage = page();  
    if (!m_inPageCache && ownerPage && (m_frame == ownerPage->mainFrame()) && hasListenerType(Document::TOUCH_LISTENER)) {  
        // Inform the Chrome Client that it no longer needs to forward touch  
        // events to WebCore as the document removes all the event listeners.  
        ownerPage->chrome()->client()->needTouchEvents(false);  
    }      
#endif  
  
    EventTarget::removeAllEventListeners();  
  
    if (DOMWindow* domWindow = this->domWindow())  
        domWindow->removeAllEventListeners();  
    for (Node* node = firstChild(); node; node = node->traverseNextNode())  
        node->removeAllEventListeners();  
} 
{% endhighlight %}
其中：
{% highlight java linenos %}
ownerPage->chrome()->client()->needTouchEvents(false); 
{% endhighlight %}
会通过JNI调用WebViewCore的needTouchEvents()，从而就会导致上面所说的shouldSkipWebKIt(d)返回ture，是的事件不会被派遣到WebKit而直接被添加到UI的消息队列。这也就是为什么在网页在Load或者切换的过程中，前面一段时间，点击页面不会响应。当然，在Document初始化完毕之后，会调用needTouchEvents(true)告诉WebViewInputDispatcher ：Come on! I need your touch. 这样，Touch输入事件就被分别派遣到Ui消息队列和WebKit消息队列。   
    
其实到这里，touch消息分拣已经很清晰简单了：   
![Alt text](/res/images/ontouchevents.png)
##WebKit消息的处理
{% highlight java linenos %}
private final DispatchEventQueue mWebKitDispatchEventQueue = new DispatchEventQueue();  
private final TouchStream mWebKitTouchStream = new TouchStream();  
private final WebKitCallbacks mWebKitCallbacks;  
private final WebKitHandler mWebKitHandler;
{% endhighlight %}
WebViewInputDispatcher定义了专门的mWebKitDispatchEventQueue来处理派发给WebKit(WebCore)的touch输入事件。enequeueWebKitEventLocked最终会将事件添加到mWebKitDispatchEvenetQueue当中，然后调用scheduleWebKitDispatchLocked()发送MSG_DISPATCH_WEBKIT_EVENTS消息给mWebKitHandler去处理事件。 
{% highlight java linenos %} 
//mWebKitHandler的handleMessage  
@Override  
public void handleMessage(Message msg) {  
    switch (msg.what) {  
        case MSG_DISPATCH_WEBKIT_EVENTS:  
            dispatchWebKitEvents(true);  
            break;  
        default:  
            throw new IllegalStateException("Unknown message type: " + msg.what);  
    }      
}
{% endhighlight %}   
dispatchWebKitEvenets()才是真正最终对mWebKitDispatcherEventQueue中消息进行分发和处理的：
{% highlight java linenos %}
private void dispatchWebKitEvents(boolean calledFromHandler) {  
     for (;;) {  
         // Get the next event, but leave it in the queue so we can move it to the UI  
         // queue if a timeout occurs.  
         DispatchEvent d;  
         MotionEvent event;  
         final int eventType;  
         int flags;  
         synchronized (mLock) {  
             if (!ENABLE_EVENT_BATCHING) {  
                 drainStaleWebKitEventsLocked();  
             }  
             d = mWebKitDispatchEventQueue.mHead;  
             if (d == null) {  
                 if (mWebKitDispatchScheduled) {  
                     mWebKitDispatchScheduled = false;  
                     if (!calledFromHandler) {  
                         mWebKitHandler.removeMessages(  
                                 WebKitHandler.MSG_DISPATCH_WEBKIT_EVENTS);  
                     }  
                 }  
                 return;  
             }  
     // ...  
     // ...  
{% endhighlight %}  
内部逻辑其实就是一个for循环，不停的取消息，然后根据当前的状态然后处理。   
##UI消息的处理：
{% highlight java linenos %}
// UI state, tracks events observed by the UI.  (guarded by mLock)  
private final DispatchEventQueue mUiDispatchEventQueue = new DispatchEventQueue();  
private final TouchStream mUiTouchStream = new TouchStream();  
private final UiCallbacks mUiCallbacks;  
private final UiHandler mUiHandler;  
private boolean mUiDispatchScheduled;
{% endhighlight %} 
跟WebKit的消息处理类似，WebViewInputDispatcher也定义了专门处理UI事件的mUiDispatcherQueue，每当有UI的Touch输入事件的时候，enqueueUiEventLocked()的事件最终会被添加到mUiDispatcherQueue中，然后调用scheduleUiDispatchLocked，发送MSG_DISPATCH_UI_EVENTS，然后触发dispatchUiEvents： 
{% highlight java linenos %} 
    /** 
     * Dispatches pending UI events. 
     * Must only be called from the UI thread. 
     * 
     * This method may be used to flush the queue of pending input events 
     * immediately.  This method may help to reduce input dispatch latency 
     * if called before certain expensive operations such as drawing. 
     */  
    public void dispatchUiEvents() {  
        dispatchUiEvents(false);  
    }  
  
    private void dispatchUiEvents(boolean calledFromHandler) {  
        for (;;) {  
            MotionEvent event;  
            final int eventType;  
            final int flags;  
            synchronized (mLock) {  
                DispatchEvent d = mUiDispatchEventQueue.dequeue();  
                if (d == null) {  
                    if (mUiDispatchScheduled) {  
                        mUiDispatchScheduled = false;  
                        if (!calledFromHandler) {  
                            mUiHandler.removeMessages(UiHandler.MSG_DISPATCH_UI_EVENTS);  
                        }  
                    }  
                    return;  
                }  
     // ...  
     }  
     // ...  
}   
{% endhighlight %} 
同样，会进入一个for循环，不停的取事件然后处理。   
篇幅有点长了，看到这里，可能还不是非常清楚，用下面这图概括下：  
![Alt text](/res/images/webviewinputdispatcher.PNG)
在所有的WebKit消息被处理的过程中，有些touch事件是需要给Ui进行反馈的，例如高亮，长按弹出菜单等等。具体这些事件会在后续文章中逐一进行解析的。



