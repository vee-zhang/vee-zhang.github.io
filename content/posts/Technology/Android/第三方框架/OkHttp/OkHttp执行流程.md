---
title: "OkHttp3解读"
subtitle: ""
date: 2021-09-07T09:49:28+08:00
lastmod: 2021-09-07T09:49:28+08:00
draft: false
authors: []
description: ""

tags: []
categories: []
series: [第三方框架]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---
回顾并且深入一下OkHttp（v3.10.0）的原理。
<!--more-->

### 用法

```kotlin
val client = OkHttpClient.Builder().build();

val request = Request.Builder()
    .url("https://wanandroid.com/wxarticle/chapters/json")
    .get()
    .build()

client.newCall(request).enqueue(responseCallback = object : Callback {

    override fun onFailure(call: Call, e: IOException) {
        val i = 0;
    }

    override fun onResponse(call: Call, response: Response) {
        val i = 0;
    }
})
```

### 构建OkHttpClient

构造方法其实就是调用了**建造者模式**。

```java
public Builder() {
    // 执行策略，同步or异步
    dispatcher = new Dispatcher();
    // 协议
    protocols = DEFAULT_PROTOCOLS;
    connectionSpecs = DEFAULT_CONNECTION_SPECS;
    // 事件监听器
    eventListenerFactory = EventListener.factory(EventListener.NONE);
    // 代理
    proxySelector = ProxySelector.getDefault();
    // cookie
    cookieJar = CookieJar.NO_COOKIES;
    // 套接字工厂
    socketFactory = SocketFactory.getDefault();
    // 域名验证
    hostnameVerifier = OkHostnameVerifier.INSTANCE;
    // 证书
    certificatePinner = CertificatePinner.DEFAULT;
    // 代理认证
    proxyAuthenticator = Authenticator.NONE;
    // 用户身份认证
    authenticator = Authenticator.NONE;

    // 初始化链接池
    connectionPool = new ConnectionPool();
    // DNS
    dns = Dns.SYSTEM;
    // 
    followSslRedirects = true;
    followRedirects = true;
    retryOnConnectionFailure = true;
    connectTimeout = 10_000;
    readTimeout = 10_000;
    writeTimeout = 10_000;
    pingInterval = 0;
  }
```

OkHttpClient主要作用就是对网络请求进行配置。

#### Dispatcher 任务分发器，同步与异步有不同的分发机制

```java
public final class Dispatcher {
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;


  private @Nullable Runnable idleCallback;

  // 线程池
  private @Nullable ExecutorService executorService;

  // 准备队列，缓存了即将被执行的异步call
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  // 异步运行队列，缓存正在运行中，或者已经取消但是还没结束的call
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  // 同步运行队列
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  // 构造方法
  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  // 默认的线程池
  // 核心线程为0，
  // 最大线程数为无限大
  // 保活时间为60秒
  // 
  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }

  // 设置最大「并发」请求数
  public synchronized void setMaxRequests(int maxRequests) {
    if (maxRequests < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequests);
    }
    this.maxRequests = maxRequests;
    promoteCalls();
  }

  public synchronized int getMaxRequests() {
    return maxRequests;
  }

  // 设置每个host的最大并发请求数
  public synchronized void setMaxRequestsPerHost(int maxRequestsPerHost) {
    if (maxRequestsPerHost < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequestsPerHost);
    }
    this.maxRequestsPerHost = maxRequestsPerHost;
    promoteCalls();
  }

  public synchronized int getMaxRequestsPerHost() {
    return maxRequestsPerHost;
  }

  // 闲置时回调
  // 当没有运行中的call时
  // 同步与异步call 有不同的闲时判定
  public synchronized void setIdleCallback(@Nullable Runnable idleCallback) {
    this.idleCallback = idleCallback;
  }

  // 同步执行
  synchronized void executed(RealCall call) {
    // 添加进队列
    runningSyncCalls.add(call);
  }

  // 异步执行
  synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      // 添加队列
      runningAsyncCalls.add(call);
      // 线程池执行
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
}
```

`Dispatcher`是策略执行器，核心是三条队列：
  ```java
  // 准备队列，缓存了即将被执行的异步call
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  // 异步运行队列，缓存正在运行中，或者已经取消但是还没结束的call
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  // 同步运行队列
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
  ```

- 同步：直接将call添加进了同步队列。
- 异步：判断当前运行中的请求数是否峰值，如果已经达到最大数，则将call放入等待队列；否则放入异步运行中队列，并且让线程池立即执行。

> 这里使用了`ArrayDeque`这个类，ArrayDeque是Java集合中双端队列的**数组实现**，相似的还有双端队列的链表实现（LinkedList）。它作为栈使用时比 Stack 类效率要高，作为队列使用时比LinkedList要快。

#### 协议 Protocol

```java
public enum Protocol {
  // http 1.0
  HTTP_1_0("http/1.0"),

  // http1.1
  HTTP_1_1("http/1.1"),

  // 不认识这个
  SPDY_3("spdy/3.1"),

  // http2.0
  HTTP_2("h2"),

  // Http3.0,谷歌基于udp实现的quic协议，解决了头部阻塞和弱网环境下网络稳定问题
  QUIC("quic");
}
```

### Request

Request是个最终类，只是用来表示请求所需的信息：

```java
public final class Request {
  final HttpUrl url;
  final String method;
  final Headers headers;
  final @Nullable RequestBody body;
  final Object tag;

  private volatile CacheControl cacheControl; // Lazily initialized.
```

### Response

```java
public final class Response implements Closeable {
  final Request request;
  final Protocol protocol;
  final int code;
  final String message;
  // 握手
  final @Nullable Handshake handshake;
  final Headers headers;
  final @Nullable ResponseBody body;
  final @Nullable Response networkResponse;
  final @Nullable Response cacheResponse;
  final @Nullable Response priorResponse;
  final long sentRequestAtMillis;
  final long receivedResponseAtMillis;

  private volatile CacheControl cacheControl; 
```

### Call 表示请求的行为过程

#### Call接口

```java
public interface Call extends Cloneable {
 
  // 返回Request
  Request request();

  // 同步执行，可直接返回Response
  Response execute() throws IOException;

  // 异步执行，需要传递回调
  void enqueue(Callback responseCallback);

  // 取消
  void cancel();

  boolean isExecuted();

  boolean isCanceled();

  // 克隆
  Call clone();

  // 工厂
  interface Factory {
    Call newCall(Request request);
  }
}
```

Call接口主要是定义了请求的行为，包含同步请求的行为和异步请求的行为，他的实现类是`RealCall`。

#### RealCall

```java
final class RealCall implements Call {

  final OkHttpClient client;

  final RetryAndFollowUpInterceptor retryAndFollowUpInterceptor;

  // 状态监听器
  private EventListener eventListener;

  final Request originalRequest;

  final boolean forWebSocket;

  private boolean executed;

  private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }

  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }

  // 同步执行
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      client.dispatcher().finished(this);
    }
  }

  // 异步执行
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }

  @Override public void cancel() {
    retryAndFollowUpInterceptor.cancel();
  }

  @Override public synchronized boolean isExecuted() {
    return executed;
  }

  @Override public boolean isCanceled() {
    return retryAndFollowUpInterceptor.isCanceled();
  }

  @SuppressWarnings("CloneDoesntCallSuperClone") // We are a final type & this saves clearing state.
  @Override public RealCall clone() {
    return RealCall.newRealCall(client, originalRequest, forWebSocket);
  }

  StreamAllocation streamAllocation() {
    return retryAndFollowUpInterceptor.streamAllocation();
  }

  final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }

    String host() {
      return originalRequest.url().host();
    }

    Request request() {
      return originalRequest;
    }

    RealCall get() {
      return RealCall.this;
    }

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }

  String redactedUrl() {
    return originalRequest.url().redact();
  }

  // 从OkhttpClient中拿出各种所需的拦截器给自己
  // 最后调用chain.proceed(originalRequest)来处理请求
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
}
```

RealCall遵循`Call`接口，表示网络请求的**行为**，包括是同步请求还是异步请求，请求成功还是失败。

在RealCall中，提供了同步执行方法execute和异步执行方法enqueue，但是其内部都是通过调用`client.dispatch.execute/enqueue`方法来实现的，所以**RealCall可以看作是OkhttpClient的代理**。

而无论哪种策略，最终都会调用`getResponseWithInterceptorChain()`方法，这个方法的流程是：
  - 创建拦截器管理器RealInterceptorChain；
  - 从client中获取拦截器；
  - 通过拦截器管理器处理原始请求。

### 拦截器 负责将Request处理为Response

```java
public interface Interceptor {

  // 拦截回调方法
  Response intercept(Chain chain) throws IOException;

  // 拦截器子类：拦截器链式管理器，实现类是RealInterceptorChain
  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    @Nullable Connection connection();

    Call call();

    int connectTimeoutMillis();

    Chain withConnectTimeout(int timeout, TimeUnit unit);

    int readTimeoutMillis();

    Chain withReadTimeout(int timeout, TimeUnit unit);

    int writeTimeoutMillis();

    Chain withWriteTimeout(int timeout, TimeUnit unit);
  }
}
```

### 拦截器管理器(拦截器链)

> 看到这，我就觉得JW大神写的代码是真牛逼啊，不服都不行，层次分明，各层都符合六大原则，看的太爽了。

```java
public final class RealInterceptorChain implements Interceptor.Chain {
  private final List<Interceptor> interceptors;
  private final StreamAllocation streamAllocation;
  private final HttpCodec httpCodec;
  private final RealConnection connection;
  // 一开始为0
  private final int index;
  private final Request request;
  private final Call call;
  private final EventListener eventListener;
  private final int connectTimeout;
  private final int readTimeout;
  private final int writeTimeout;

  // 初始为0
  private int calls;

  // 注意看这个构造方法
  private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;

    // 直接初始化了一个重试拦截器
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
  }

  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }

  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,RealConnection connection) throws IOException {

    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // 注意这里的index+1
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);

    Response response = interceptor.intercept(next);

    return response;
  }
}
```

>这个东西其实应该叫做「真正的拦截器链」，而我为了明确其作用，叫做「拦截器管理器」。

作用就是让队列中的拦截器one by one的处理请求，上一个拦截器负责处理下一个拦截器的返回结果，所以调用是从ArrayList的首个元素开始向下遍历到队尾，再逐渐pop回来。

我们把每一个RealInterceptorChain看作链条中的一环，每一环的`processd()`方法中，通过指定`index+1`来创建下一环，之后通过index拿到对应的拦截器，靠拦截器去处理下一环，如此递归调用，最后层层返回response。

这样做的好处是什么呢，当然是**贴合http的请求过程**。但是问题也很明显，**方法调用层级过深，并且request的处理逻辑会与response的逻辑处理耦合在一起，变得密不可分**。

> 其实以前用OK的时候就有这个感觉了，并且后来写Flutter的时候用到了dio框架，它的拦截器把request、response、error分开处理，结构更加的清晰。

那么我觉得，对于自身的`interceptors`来说，最后添加的拦截器是最先处理response的，也就是建立连接的始作俑者。


```java
interceptors.addAll(client.interceptors());
interceptors.add(retryAndFollowUpInterceptor);
interceptors.add(new BridgeInterceptor(client.cookieJar()));
interceptors.add(new CacheInterceptor(client.internalCache()));
interceptors.add(new ConnectInterceptor(client));
if (!forWebSocket) {
  interceptors.addAll(client.networkInterceptors());
}
// CallServerInterceptor拦截器用来建立连接
interceptors.add(new CallServerInterceptor(forWebSocket)); 
```

然而在CallServerInterceptor中我找不到任何建立连接的代码。通过观察，我发现这个`retryAndFollowUpInterceptor`很有意思，于是接下来要去看看它是做什么用的。

### 事件监听器eventListener

```java
public abstract class EventListener {

  /// 空的监听器
  public static final EventListener NONE = new EventListener() {
  };

  /// 静态工厂方法
  static EventListener.Factory factory(final EventListener listener) {
    
    return new EventListener.Factory() {
      // 我就不明白，这里传了个Call根本就没用上
      public EventListener create(Call call) {
        return listener;
      }
    };
  }

  // 当call被execute或者enqueue时回调，但是此时可能request还没有执行
  public void callStart(Call call) {
  }

  //当调用 Dns.lookup(String)时回调，可能回调多次
  public void dnsStart(Call call, String domainName) {
  }

  //当dnsStart之后回调
  public void dnsEnd(Call call, String domainName, List<InetAddress> inetAddressList) {
  }

  //当Socket链接建立后回调
  public void connectStart(Call call, InetSocketAddress inetSocketAddress, Proxy proxy) {
  }

  //当加密（TSL）链接开始时调用，
  public void secureConnectStart(Call call) {
  }

  // 加密结束后调用
  public void secureConnectEnd(Call call, @Nullable Handshake handshake) {
  }

  //Socket尝试建立连接之后调用
  public void connectEnd(Call call, InetSocketAddress inetSocketAddress, Proxy proxy,
      @Nullable Protocol protocol) {
  }

  // 链接失败
  public void connectFailed(Call call, InetSocketAddress inetSocketAddress, Proxy proxy,
      @Nullable Protocol protocol, IOException ioe) {
  }

  // 链接建立后回调
  public void connectionAcquired(Call call, Connection connection) {
  }

  // 链接释放后回调
  public void connectionReleased(Call call, Connection connection) {
  }

  // 请求head开始
  public void requestHeadersStart(Call call) {
  }

  // 请求head结束
  public void requestHeadersEnd(Call call, Request request) {
  }

  // 请求body开始
  public void requestBodyStart(Call call) {
  }

  /// 请求body结束
  public void requestBodyEnd(Call call, long byteCount) {
  }

  // 响应head开始
  public void responseHeadersStart(Call call) {
  }

  // 响应head结束
  public void responseHeadersEnd(Call call, Response response) {
  }

  // 响应body开始
  public void responseBodyStart(Call call) {
  }

  // 响应body结束
  public void responseBodyEnd(Call call, long byteCount) {
  }

  // call完成
  public void callEnd(Call call) {
  }

  // call 失败
  public void callFailed(Call call, IOException ioe) {
  }

  
  public interface Factory {
    
    EventListener create(Call call);
  }
}
```

事件监听器，用来侦听一次请求中经历的各种状态。


### NamedRunnable

```java
public abstract class NamedRunnable implements Runnable {
  protected final String name;

  public NamedRunnable(String format, Object... args) {
    this.name = Util.format(format, args);
  }
  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}
```
