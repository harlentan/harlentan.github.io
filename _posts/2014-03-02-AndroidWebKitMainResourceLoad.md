---
layout: post
title: Android WebKit HTML主资源加载过程
tags: [blog]
keywords: Android WebKit HTML主资源加载过程, SEO, fenesky, android, WebKit，谭海燕, HTML主资源加载, HTML主资源
description: Android WebKit HTML主资源加载过程, 详细讲解了Android WebKit HTML主资源加载的全过程
comments: true
---

##前言
在浏览器里面输入网址，最终浏览器会调用WebView的loadUrl()，然后就开始加载整个网页。整个加载过程中，最重要的一步就是HTML主资源的加载。WebKit将网页的资源分为主资源(MainResource)和子资源(SubResource)。   

<!--more-->   




##WebKit资源分类
* 主资源：HTML文件。
* 子资源：CSS, JS, JPG等等，除了HTML文件之外的所有资源都称之为子资源   
本章主要讲主资源的加载过程，子资源的加载过程后期会专门详细的分析和讲解。    

##主资源请求   
###LoadUrl
主资源的请求是从WebView的loadUrl开始的。根据之前[《Android WebKit消息处理》](http://www.fenesky.com/blog/2014/02/11/Android-WebKit-MsgHandle.html)的讲解，WebView的操作都会有WebViewClassic进行代理。资源加载肯定是由WebCore来处理的，所以，WebVewClassic会发消息给WebViewCore，让WebViewCore最终将loadUrl传递给C++层的WebKit处理：   
{% highlight java linenos %}
/** 
 * See {@link WebView#loadUrl(String, Map)} 
 */  
@Override  
public void loadUrl(String url, Map<String, String> additionalHttpHeaders) {  
    loadUrlImpl(url, additionalHttpHeaders);  
}  
  
private void loadUrlImpl(String url, Map<String, String> extraHeaders) {  
    switchOutDrawHistory();  
    WebViewCore.GetUrlData arg = new WebViewCore.GetUrlData();  
    arg.mUrl = url;  
    arg.mExtraHeaders = extraHeaders;  
    mWebViewCore.sendMessage(EventHub.LOAD_URL, arg);  
    clearHelpers();  
} 
{% endhighlight %}
WebViewCore在接收到LOAD_URL之后，会通过BrowserFrame调用nativeLoadUrl，这个BrowserFrame与C++层的mainFrame对接。这里顺便提一下clearHeapers()的作用：如果当前网页有对话框dialog，有输入法之类的，clearHelpers就是用来清理这些东西的。这也是为什么加载一个新页面的时候，但当前页面的输入法以及dialog消失等等。WebViewCore收到消息之后，会直接让BrowserFrame调用JNI:  nativeLoadUrl():
{% highlight java linenos %}
// BrowserFrame.java  
    public void loadUrl(String url, Map<String, String> extraHeaders) {  
        mLoadInitFromJava = true;  
        if (URLUtil.isJavaScriptUrl(url)) {  
            // strip off the scheme and evaluate the string  
            stringByEvaluatingJavaScriptFromString(  
                    url.substring("javascript:".length()));  
        } else {  
            /** M: add log */  
            Xlog.d(XLOGTAG, "browser frame loadUrl: " + url);  
            nativeLoadUrl(url, extraHeaders);  
        }  
        mLoadInitFromJava = false;  
    }
{% endhighlight %}
由于LoadUrl()不仅可以Load一个url，还可以执行一段js。如果load的是一段js，js并没有被继续往下load，而是直接在这里执行掉。stringByEvaluatingJavaScriptFromString也会通过jni调用v8的接口去在mainFrame的scriptController中执行，关于js在WebKit后期会专门写一篇关于WebKit的js的文章进行专门分析。到目前为止，LoadUrl还只是简单的使用一个String传递字符串而已。
{% highlight java linenos %}
// WebCoreFrameBridge.cpp  
static void LoadUrl(JNIEnv *env, jobject obj, jstring url, jobject headers)  
{  
    WebCore::Frame* pFrame = GET_NATIVE_FRAME(env, obj);  
    ALOG_ASSERT(pFrame, "nativeLoadUrl must take a valid frame pointer!");  
  
    WTF::String webcoreUrl = jstringToWtfString(env, url);  
    WebCore::KURL kurl(WebCore::KURL(), webcoreUrl);  
    WebCore::ResourceRequest request(kurl);  
    if (headers) {  
        // dalvikvm will raise exception if any of these fail  
        jclass mapClass = env->FindClass("java/util/Map");  
        jmethodID entrySet = env->GetMethodID(mapClass, "entrySet",  
                "()Ljava/util/Set;");  
        jobject set = env->CallObjectMethod(headers, entrySet);  
  
        jclass setClass = env->FindClass("java/util/Set");  
        jmethodID iterator = env->GetMethodID(setClass, "iterator",  
                "()Ljava/util/Iterator;");  
        jobject iter = env->CallObjectMethod(set, iterator);  
  
        jclass iteratorClass = env->FindClass("java/util/Iterator");  
        jmethodID hasNext = env->GetMethodID(iteratorClass, "hasNext", "()Z");  
        jmethodID next = env->GetMethodID(iteratorClass, "next",  
                "()Ljava/lang/Object;");  
        jclass entryClass = env->FindClass("java/util/Map$Entry");  
        jmethodID getKey = env->GetMethodID(entryClass, "getKey",  
                "()Ljava/lang/Object;");  
        jmethodID getValue = env->GetMethodID(entryClass, "getValue",  
                "()Ljava/lang/Object;");  
  
        while (env->CallBooleanMethod(iter, hasNext)) {  
            jobject entry = env->CallObjectMethod(iter, next);  
            jstring key = (jstring) env->CallObjectMethod(entry, getKey);  
            jstring value = (jstring) env->CallObjectMethod(entry, getValue);  
            request.setHTTPHeaderField(jstringToWtfString(env, key), jstringToWtfString(env, value));  
            env->DeleteLocalRef(entry);  
            env->DeleteLocalRef(key);  
            env->DeleteLocalRef(value);  
        }  
    // ...  
    pFrame->loader()->load(request, false);  
}
{% endhighlight %}
接下来，在JNI的LoadUrl中就开始创建ResourceRequest，由于WebView的java层面可以对url的请求头进行设定，然后通过FrameLoader进行加载。这里的pFrame就是与Java层的BrowserFrame对应的mainFrame。HTML在WebKit的层次上看，最低层的是Frame，然后才有Document，也就意味着HTML Document也是通过Frame的FrameLoader加载的：
pFrame->loader()->load(request, false);  
调用栈
最后的这句话就是让FrameLoader去加载url的request。后面的调用栈依次是：
{% highlight java linenos %}
void FrameLoader::load(const ResourceRequest& request, bool lockHistory)  
void FrameLoader::load(const ResourceRequest& request, const SubstituteData& substituteData, bool lockHistory)  
void FrameLoader::load(DocumentLoader* newDocumentLoader)  
void FrameLoader::loadWithDocumentLoader(DocumentLoader* loader, FrameLoadType type, PassRefPtr<FormState> prpFormState)  
void FrameLoader::callContinueLoadAfterNavigationPolicy(void* argument,  
    const ResourceRequest& request, PassRefPtr<FormState> formState, bool shouldContinue)  
void FrameLoader::continueLoadAfterNavigationPolicy(const ResourceRequest&, PassRefPtr<FormState> formState, bool shouldContinue)  
void FrameLoader::continueLoadAfterWillSubmitForm() 
{% endhighlight %} 
其中加载Document的DocumentLoader在load中创建的：
{% highlight java linenos %}
void FrameLoader::load(const ResourceRequest& request, const SubstituteData& substituteData, bool lockHistory)  
{  
    if (m_inStopAllLoaders)  
        return;  
          
    // FIXME: is this the right place to reset loadType? Perhaps this should be done after loading is finished or aborted.  
    m_loadType = FrameLoadTypeStandard;  
    RefPtr<DocumentLoader> loader = m_client->createDocumentLoader(request, substituteData);  
    if (lockHistory && m_documentLoader)  
        loader->setClientRedirectSourceForHistory(m_documentLoader->didCreateGlobalHistoryEntry() ? m_documentLoader->urlForHistory().string() : m_documentLoader->clientRedirectSourceForHistory());  
    load(loader.get());  
}  
{% endhighlight %} 
m\_client->createDocumentLoader(request, substituteData);中的m\_client是FrameLoaderClientAndroid。后面资源下载还有跟这个m_client打交道。在void FrameLoader::continueLoadAfterWillSubmitForm()之前，还没有真正涉及到主资源的加载，还都只是在对当前需要加载的Url进行一些列的判断，一方面是安全问题，SecurityOrigin会对Url进行安全检查，例如跨域。另一方面是Scroll，因为有时候后LoadUrl加载的Url会带有Url Fragment也就是hash。关于url的hash的内容请参考《Fragment URLS》由于URL的hash，只会滚动到页面的某一个位置，所以这种情况下也不需要真正的去请求mainResource. 如果这些检查都过了，就需要开始去加载mainResource了：
{% highlight java linenos %}
// FrameLoader.cpp  
void FrameLoader::continueLoadAfterWillSubmitForm()  
{  
    // ...  
    m_provisionalDocumentLoader->timing()->navigationStart = currentTime();  
  
    // ...  
    if (!m_provisionalDocumentLoader->startLoadingMainResource(identifier))  
        m_provisionalDocumentLoader->updateLoading();  
}
{% endhighlight %}
startLoadingMainResource这就开始load主资源也就是前面说的html文件。
三种DocumentLoader
这里需要对m_provisionalDocumentLoader进行讲解下：
{% highlight java linenos %}
RefPtr<DocumentLoader> m_documentLoader;  
RefPtr<DocumentLoader> m_provisionalDocumentLoader;  
RefPtr<DocumentLoader> m_policyDocumentLoader;  
void setDocumentLoader(DocumentLoader*);  
void setPolicyDocumentLoader(DocumentLoader*);  
void setProvisionalDocumentLoader(DocumentLoader*);
{% endhighlight %}
我们可以看到在FrameLoader.h中定义了三个DocumentLoader，WebKit其实是按角色划分这几个DocumentLoader的。其中：m\_documentLoader是上一次已经加载过的DocumentLoader的指针，m\_policyDocumentLoader就是用来做一些策略性的工作的，例如延迟加载等等。m\_provisionalDocumentLoade是用来做实际的加载工作的。当一个DocumentLoader的工作完成之后，会通过setXXXXDocumentLoader来传递指针。按照URL加载的主流程：PolicyChcek------>Load MainResouce。也就是先进行策略检查，最后才开始加载主资源。那么这个三个DocumentLoader的顺序应该是先createDocumentLoader后的指针传递给m\_pollicyDocumentLoader，在策略检查完之后，将指针传递给m\_provisionalDocumentLoader，在Document加载完毕之后，将指针传递给m_documentLoader。
{% highlight java linenos %}
// FrameLoader.cpp  
void FrameLoader::loadWithDocumentLoader(DocumentLoader* loader, FrameLoadType type, PassRefPtr<FormState> prpFormState)  
{  
    // ...     
    policyChecker()->stopCheck();  
    // ...  
    setPolicyDocumentLoader(loader);  
    // ..  
}  
  
void FrameLoader::continueLoadAfterNavigationPolicy(const ResourceRequest&, PassRefPtr<FormState> formState, bool shouldContinue)  
{  
    // ...  
    setProvisionalDocumentLoader(m_policyDocumentLoader.get());  
    m_loadType = type;  
    setState(FrameStateProvisional);  
    // ...  
    setPolicyDocumentLoader(0);  
  
}  
  
void FrameLoader::transitionToCommitted(PassRefPtr<CachedPage> cachedPage)  
{  
    // ...  
    setDocumentLoader(m_provisionalDocumentLoader.get());  
    setProvisionalDocumentLoader(0);  
    // ...  
}  
  
void FrameLoader::checkLoadCompleteForThisFrame()  
{  
    switch (m_state) {  
        case FrameStateProvisional: {  
                // ...  
  
                // If we're in the middle of loading multipart data, we need to restore the document loader.  
                if (isReplacing() && !m_documentLoader.get())  
                    setDocumentLoader(m_provisionalDocumentLoader.get());  
  
                // Finish resetting the load state, but only if another load hasn't been started by the  
                // delegate callback.  
                if (pdl == m_provisionalDocumentLoader)  
                    clearProvisionalLoad();  
                  
    }  
  
    // ...  
}  
{% endhighlight %}
上面代码片段可以看出，这三个DocumentLoader的承接关系是一环扣一环。由于index.html加载在WebKit中分为2中方式：如果是前进后退，index.html是从CachedPage中加载的，FrameLoader::transitionToCommitted就是在从CachedPage中加载完成之后被调用的，void FrameLoader::checkLoadCompleteForThisFrame()这是在从网络加载完成之后被调用的。
{% highlight java linenos %}
void FrameLoader::recursiveCheckLoadComplete()  
{  
    Vector<RefPtr<Frame>, 10> frames;  
      
    for (RefPtr<Frame> frame = m_frame->tree()->firstChild(); frame; frame = frame->tree()->nextSibling())  
        frames.append(frame);  
      
    unsigned size = frames.size();  
    for (unsigned i = 0; i < size; i++)  
        frames[i]->loader()->recursiveCheckLoadComplete();  
      
    checkLoadCompleteForThisFrame();  
}  
  
// Called every time a resource is completely loaded, or an error is received.  
void FrameLoader::checkLoadComplete()  
{  
    ASSERT(m_client->hasWebView());  
      
    m_shouldCallCheckLoadComplete = false;  
  
    // FIXME: Always traversing the entire frame tree is a bit inefficient, but   
    // is currently needed in order to null out the previous history item for all frames.  
    if (Page* page = m_frame->page())  
        page->mainFrame()->loader()->recursiveCheckLoadComplete();  
}  
{% endhighlight %}
需要强调的是，WebKit需要对Page里面的所有Frame进行确认加载完毕之后，最后将setDocumentLoader()。对于这一点我个人理解是还有优化的空间。
###startLoadingMainResource
在m_provisionalDocumentLoader调用startLoadingMainResource之后，就开始准备发送网络请求了。调用栈如下：
{% highlight java linenos %}
bool DocumentLoader::startLoadingMainResource(unsigned long identifier)  
bool MainResourceLoader::load(const ResourceRequest& r, const SubstituteData& substituteData)  
bool MainResourceLoader::loadNow(ResourceRequest& r)  
PassRefPtr<ResourceHandle> ResourceHandle::create(NetworkingContext* context,   
    const ResourceRequest& request,  
    ResourceHandleClient* client,  
    bool defersLoading,  
    bool shouldContentSniff)  
bool ResourceHandle::start(NetworkingContext* context)  
PassRefPtr<ResourceLoaderAndroid> ResourceLoaderAndroid::start(  
        ResourceHandle* handle, const ResourceRequest& request,  
    FrameLoaderClient* client, bool isMainResource, bool isSync)  
bool WebUrlLoaderClient::start(bool isMainResource, bool isMainFrame, bool sync, WebRequestContext* context) 
{% endhighlight %}
需要指出的是，虽然LoadUrl最后是在WebCore线程中执行的，但是最后资源下载是在Chromium\_net的IO线程中进行的。在资源下载完毕之后，网络数据会交给FrameLoaderClientAndroid
网络数据
Android WebKit数据下载在Chromium\_net的IO线程中完成之后会通过WebUrlLoaderClient向WebCore提交数据。WebKt的调用栈如下：
{% highlight C++ %}
// Finish  
void WebUrlLoaderClient::didFinishLoading()  
void ResourceLoader::didFinishLoading(ResourceHandle*, double finishTime)  
void MainResourceLoader::didFinishLoading(double finishTime)  
void FrameLoader::finishedLoading()  
void DocumentLoader::finishedLoading()  
void FrameLoader::finishedLoadingDocument(DocumentLoader* loader)  
void FrameLoaderClientAndroid::finishedLoading(DocumentLoader* docLoader)  
void FrameLoaderClientAndroid::committedLoad(DocumentLoader* loader,   
     const char* data, int length)  
void DocumentLoader::commitData(const char* bytes, int length)  
  
  
  
  
// Receive Data  
void WebUrlLoaderClient::didReceiveData(scoped_refptr<net::IOBuffer> buf, int size)  
void ResourceLoader::didReceiveData(ResourceHandle*, const char* data, int length,   
     int encodedDataLength)  
void ResourceLoader::didReceiveData(const char* data, int length,  
     long long encodedDataLength, bool allAtOnce)  
void MainResourceLoader::addData(const char* data, int length, bool allAtOnce)  
void DocumentLoader::receivedData(const char* data, int length)  
void DocumentLoader::commitLoad(const char* data, int length)  
void FrameLoaderClientAndroid::committedLoad(DocumentLoader* loader,  
     const char* data, int length)  
void DocumentLoader::commitData(const char* bytes, int length) 
{% endhighlight %}
这个过程其实分为两步，一步是Chromium\_net收到数据，另一部是Chromium_net通知WebKit，数据已经下载完毕可以finish了。这个两个过程都会调用FrameLoaderClienetAndroid::committedLoad()。只不过参数不一样，在finish的时候，将传入的length为0，这样通知WebKit，数据已经传送完毕，记者WebKit就开始使用commitData拿到的数据进行解析，构建Dom Tree和Render Tree。关于Dom Tree Render Tree的构建过程下一节详细的讲述。