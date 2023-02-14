---
title: "WebView性能优化"
subtitle: ""
date: 2022-01-06T09:29:17+08:00
lastmod: 2022-01-06T09:29:17+08:00
draft: false
authors: []
description: ""

tags: []
categories: []
series: []

---

<!--more-->

### WebView与原生对比差在哪里？

这里引用百度APP图片来说明。

![WebView加载及渲染过程](WebView加载过程.awebp)

百度的开发人员将这一整个过程划分为了四个阶段，并统计出了各个阶段的平均耗时。
可以看到，在初始化组件阶段就花费了 260 ms，首次创建耗时均值为 500 ms，毫无疑问这是我们要优化的第一点。而最耗时的当属正文加载&渲染和图片加载两个阶段。为什么会这么耗时呢，因为这两个阶段需要进行多次网络请求、JS 调用、IO 读写。所以这里也是我们需要优化的地方。
可以得出优化方向：

- WebView预创建和复用
- 渲染优化（JS、CSS、图片）
- 模板优化（拆分、预热、复用）

### WebView预创建和复用

WebView 的创建是比较耗时的，首次创建耗时几百毫秒，因此预创建和复用尤为重要。
大致逻辑是先创建WebView并缓存起来，等到需要的时候直接取出来，代码如下：

```kotlin
class WebViewManager private constructor() {

    // 为了阅读体验，省略部分代码

    private val webViewCache: MutableList<WebView> = ArrayList(1)

    private fun create(context: Context): WebView {
        val webView = WebView(context)
	// ......
        return webView
    }

    fun prepare(context: Context) {
        if (webViewCache.isEmpty()) {
            Looper.myQueue().addIdleHandler {
                webViewCache.add(create(MutableContextWrapper(context)))
                false
            }
        }
    }

    fun obtain(context: Context): WebView {
        if (webViewCache.isEmpty()) {
            webViewCache.add(create(MutableContextWrapper(context)))
        }
        val webView = webViewCache.removeFirst()
        val contextWrapper = webView.context as MutableContextWrapper
        contextWrapper.baseContext = context
        webView.clearHistory()
        webView.resumeTimers()
        return webView
    }

    fun recycle(webView: WebView) {
        try {
            webView.stopLoading()
            webView.loadDataWithBaseURL(null, "", "text/html", "utf-8", null)
            webView.clearHistory()
            webView.pauseTimers()
            webView.webChromeClient = null
            webView.webViewClient = null
            val parent = webView.parent
            if (parent != null) {
                (parent as ViewGroup).removeView(webView)
            }
        } catch (e: Exception) {
            e.printStackTrace()
        } finally {
            if (!webViewCache.contains(webView)) {
                webViewCache.add(webView)
            }
        }
    }

    fun destroy() {
        try {
            webViewCache.forEach {
                it.removeAllViews()
                it.destroy()
                webViewCache.remove(it)
            }
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }

}
```

这里需要注意以下几点：

- 预加载时机的选择
    
    WebView 的创建是比较耗时的，为了使预加载不会影响到当前主线程任务，我们选择 IdleHandler 来执行预创建，以保证不会影响到当前主线程任务。详细请看 prepare(context: Context) 方法。

- Context的选择

    每个 WebView 需要和对应的 Activity Context 实例进行绑定，为了保证预加载的 WebView Context 和最终的 Context 之间的一致性，我们通过 MutableContextWrapper 来解决这个问题。MutableContextWrapper 允许外部替换它的 baseContext ，因此 prepare(context: Context)方法可以传 applicationContext 进行预创建，等到实际调用时再进行替换，详细请看 obtain(context: Context) 方法。

- 复用和销毁

    在页面关闭前调用 recycle(webView: WebView) 进行回收，在应用退出前调用 destroy() 进行销毁。

- 复用WebView返回空白

    在调用 recycle(webView: WebView) 进行回收时，我们会调用 loadDataWithBaseURL(null, "", "text/html", "utf-8", null) 清除页面内容，导致复用时的加载栈底就是这个空白页面，所以我们需要在返回时对栈底进行判断，如果为空则直接返回，代码如下：

    ```kotlin
    fun canGoBack(): Boolean {
    val canBack = webView.canGoBack()
    if (canBack) webView.goBack()
    val backForwardList = webView.copyBackForwardList()
    val currentIndex = backForwardList.currentIndex
    if (currentIndex == 0) {
        val currentUrl = backForwardList.currentItem.url
        val currentHost = Uri.parse(currentUrl).host
        //栈底不是链接则直接返回
        if (currentHost.isNullOrBlank()) return false
    }
    return canBack
}
```

### 渲染优化（JS、CSS、图片）

WebView 在加载内容的时候会进行多次网络请求、JS 调用、IO 读写。我们可以借由内核的 shouldInterceptRequest 回调，拦截资源请求由客户端进行下载，并以管道方式填充到内核的 WebResourceResponse中，这里引用百度APP图片来说明。

![WebView渲染过程](WebView渲染过程.awebp)

预置离线包

精简并抽取公共的 JS 和 CSS 文件作为通用资源，将抽取的资源存放在 assets 下，再通过约定的规则去匹配，代码如下：

```kotlin
webView.webViewClient = object : WebViewClient() {

    // 为了阅读体验，省略部分代码

    override fun shouldInterceptRequest(
        view: WebView?,
        request: WebResourceRequest?
    ): WebResourceResponse? {
        if (view != null && request != null) {
            if(canAssetsResource(request)){
                return assetsResourceRequest(view.context, request)
            }
        }
        return super.shouldInterceptRequest(view, request)
    }
}

private fun assetsResourceRequest(
    context: Context, 
    webRequest: WebResourceRequest
): WebResourceResponse? {

    ...
    
    try {
        val url = webRequest.url.toString()
        val filenameIndex = url.lastIndexOf("/") + 1
        val filename = url.substring(filenameIndex)
        val suffixIndex = url.lastIndexOf(".")
        val suffix = url.substring(suffixIndex + 1)
        val webResourceResponse = WebResourceResponse()
        webResourceResponse.mimeType = getMimeTypeFromUrl(url)
        webResourceResponse.encoding = "UTF-8"
        webResourceResponse.data = context.assets.open("$suffix/$filename")
        return webResourceResponse
    } catch (e: Exception) {
        e.printStackTrace()
    }
    return null
}
```

### 接口更新缓存资源

除了预置离线包外，我们还可以通过接口请求，获取最新的缓存资源，以及通过请求资源的类型自行缓存，代码如下：

```kotlin
webView.webViewClient = object : WebViewClient() {

   ...

    override fun shouldInterceptRequest(
        view: WebView?,
        request: WebResourceRequest?
    ): WebResourceResponse? {
        if (view != null && request != null) {
            if(canCacheResource(request)){
                return cacheResourceRequest(view.context, request)
            }
        }
        return super.shouldInterceptRequest(view, request)
    }
}

private fun canCacheResource(webRequest: WebResourceRequest): Boolean {

    // 为了阅读体验，省略部分代码

    val url = webRequest.url.toString()
    val extension = getExtensionFromUrl(url)
    return extension == "ico" || extension == "bmp" || extension == "gif"
            || extension == "jpeg" || extension == "jpg" || extension == "png"
            || extension == "svg" || extension == "webp" || extension == "css"
            || extension == "js" || extension == "json" || extension == "eot"
            || extension == "otf" || extension == "ttf" || extension == "woff"
}

private fun cacheResourceRequest(
    context: Context, 
    webRequest: WebResourceRequest
): WebResourceResponse? {

    ...
    
    try {
    	val url = webRequest.url.toString()
    	val cachePath = CacheUtils.getCacheDirPath(context, "web_cache")
    	val filePathName = cachePath + File.separator + url.encodeUtf8().md5().hex()
    	val file = File(filePathName)
    	if (!file.exists() || !file.isFile) {
    		runBlocking {
                        // 开启网络请求下载资源
    			download(HttpRequest(url).apply {
    				webRequest.requestHeaders.forEach { putHeader(it.key, it.value) }
    			}, filePathName)
    		}
    	}
    	if (file.exists() && file.isFile) {
    		val webResourceResponse = WebResourceResponse()
    		webResourceResponse.mimeType = getMimeTypeFromUrl(url)
    		webResourceResponse.encoding = "UTF-8"
    		webResourceResponse.data = file.inputStream()
    		webResourceResponse.responseHeaders = mapOf("access-control-allow-origin" to "*")
    		return webResourceResponse
    	}
    } catch (e: Exception) {
    	e.printStackTrace()
    }
    return null
}
```

我们通过canCacheResource(webRequest: WebResourceRequest)来判断是否是需要缓存的资源。
再根据URL去获取缓存中文件，否则开启网络请求下载资源，详细请看 cacheResourceRequest(context: Context, webRequest: WebResourceRequest) 。
这边仅对图片、字体、CSS、JS、JSON进行缓存，可根据项目实际情况缓存更多类型资源。

### 模板优化（拆分、预热、复用）

关于模板，代码如下：

```html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
	<title>title</title>
        <link rel="stylesheet" type="text/css" href="xxx.css">
	<script>
		function changeContent(data){
			document.getElementById('content').innerHTML=data;
		}
	</script>
</head>
<body>
    <div id="content"></div>
</body>
</html>
```

客户端加载模板代码（温馨提示：上面只是例子，实际模板根据情况拆分），加载完成后再调用 JS 方法注入数据。


数据哪里来呢？这里以列表页跳转详情页举个例子，仅供参考：

列表页接口返回列表数据的时候带上详情内容，跳转详情页的时候带上内容数据。优点简单粗暴，缺点耗费流量。

当然还有其他方法这里不再多说，可根据自己的实际需求进行选择。

