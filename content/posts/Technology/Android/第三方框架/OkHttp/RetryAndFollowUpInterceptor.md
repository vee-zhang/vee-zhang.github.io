---
title: "RetryAndFollowUpInterceptor"
subtitle: "重试和重定向拦截器"
date: 2021-09-07T17:45:01+08:00
lastmod: 2021-09-07T17:45:01+08:00
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

### RealCall中的传值

```java
private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
}
```

RealCall初始化时，会创建这个`RetryAndFollowUpInterceptor`拦截器，并且传递了`OkHttpClient`。

观察RealCall中的所有默认拦截器，RealCall对于这个拦截器的处理优先级是最高的，甚至**在构造方法中就初始化**了，而且在众多内置拦截器中，**RealCall单只给`RetryAndFollowUpInterceptor`传递了OkhttpClient**，可见这个拦截器的重要性！

### RetryAndFollowUpInterceptor的处理过程

```java
public final class RetryAndFollowUpInterceptor implements Interceptor {

  private static final int MAX_FOLLOW_UPS = 20;

  private final OkHttpClient client;
  private final boolean forWebSocket;
  private volatile StreamAllocation streamAllocation;
  private Object callStackTrace;
  private volatile boolean canceled;

  public RetryAndFollowUpInterceptor(OkHttpClient client, boolean forWebSocket) {
    this.client = client;
    this.forWebSocket = forWebSocket;
  }

  // 取消，不需要看了
  public void cancel() {
    canceled = true;
    StreamAllocation streamAllocation = this.streamAllocation;
    if (streamAllocation != null) streamAllocation.cancel();
  }

  @Override public Response intercept(Chain chain) throws IOException {
    
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();

    // 初始化数据流分配器
    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);

    this.streamAllocation = streamAllocation;

    // 重定向次数
    int followUpCount = 0;

    // 前一个response
    Response priorResponse = null;

    // 又是个永动机
    while (true) {

      Response response;
      boolean releaseConnection = true;
      try {
        // 调用下一个拦截器，最终获取到response
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        //发现异常就重试
        if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        //发现异常就重试
        ...
        continue;
      } finally {
        // 释放连接
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      // 附加前一个response，这种都是没有body
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)//没有body
                    .build())
            .build();
      }

      // 根据上一个response，生成重定向需要的request
      Request followUp = followUpRequest(response, streamAllocation.route());

      // 如果不再需要重定向，就直接返回response了
      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      // 关闭 response的body
      closeQuietly(response.body());

      // 如果重定向/重试次数大于20次就会报错
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }
      ...

      // 如果不是同一个主机地址，则连接不能复用，只能释放上一个，并且新建一个连接；否则将复用之前创建好的连接
      if (!sameConnection(response, followUp.url())) {
        // 用request重建连接
        streamAllocation.release();
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;
      } 

      request = followUp;
      priorResponse = response;
    }
  }

  // 拼装地址，包含主机、端口号、DNS等等
  private Address createAddress(HttpUrl url) {
    SSLSocketFactory sslSocketFactory = null;
    HostnameVerifier hostnameVerifier = null;
    CertificatePinner certificatePinner = null;
    // 判断是否https
    if (url.isHttps()) {
      sslSocketFactory = client.sslSocketFactory();
      hostnameVerifier = client.hostnameVerifier();
      certificatePinner = client.certificatePinner();
    }

    return new Address(url.host(), url.port(), client.dns(), client.socketFactory(),
        sslSocketFactory, hostnameVerifier, certificatePinner, client.proxyAuthenticator(),
        client.proxy(), client.protocols(), client.connectionSpecs(), client.proxySelector());
  }

  // 尝试从与服务器通信失败中恢复（重试）
  private boolean recover(IOException e, StreamAllocation streamAllocation,
      boolean requestSendStarted, Request userRequest) {

    streamAllocation.streamFailed(e);

    // 重试
    // return false - 应用层拒绝了重试,这是我们在初始化client时设置的
    if (!client.retryOnConnectionFailure()) return false;
    ...
    // 如果有更多的路由，比如DNS返回了多个CDN的ip地址，就切换线路重试，否则取消。
    if (!streamAllocation.hasMoreRoutes()) return false;
    ...
    return true;
  }

  // 判断连接是否可以恢复
  private boolean isRecoverable(IOException e, boolean requestSendStarted) {
    ...
  }

  // 根据responseCode判断，如果response需要重定向，则创建一个重定向所需的request
  private Request followUpRequest(Response userResponse, Route route) throws IOException {
    
    int responseCode = userResponse.code();

    final String method = userResponse.request().method();

    switch (responseCode) {

      ...
      case HTTP_UNAUTHORIZED:// OAuth2.0 授权码方式要求重定向到一个授权页面
        return client.authenticator().authenticate(route, userResponse);

      ...
      
      // 响应码是3xx系列就需要重定向处理
      case HTTP_MULT_CHOICE:
      case HTTP_MOVED_PERM:
      case HTTP_MOVED_TEMP:
      case HTTP_SEE_OTHER:
        // 客户端允许重定向吗？
        if (!client.followRedirects()) return null;

        // 原来response的head中有这么个东东
        String location = userResponse.header("Location");
        if (location == null) return null;

        // 根据location生成重定向的url
        // 这个resolve方法是真复杂，懒得看了
        HttpUrl url = userResponse.request().url().resolve(location);

        // 不会重定向到不支持的协议
        if (url == null) return null;

        // 不会在http和https之间重定向
        boolean sameScheme = url.scheme().equals(userResponse.request().url().scheme());
        if (!sameScheme && !client.followSslRedirects()) return null;

        // 大多数的重定向都不需要requestBody
        // 其实这里就是依据条件来分别调用建造者模式，最终会构建一个符合条件的request
        Request.Builder requestBuilder = userResponse.request().newBuilder();
        
        if (HttpMethod.permitsRequestBody(method)) {
          final boolean maintainBody = HttpMethod.redirectsWithBody(method);
          if (HttpMethod.redirectsToGet(method)) {
            requestBuilder.method("GET", null);
          } else {
            RequestBody requestBody = maintainBody ? userResponse.request().body() : null;
            requestBuilder.method(method, requestBody);
          }
          if (!maintainBody) {
            requestBuilder.removeHeader("Transfer-Encoding");
            requestBuilder.removeHeader("Content-Length");
            requestBuilder.removeHeader("Content-Type");
          }
        }

        // 如果重定向的不是之前的连接，就移除OAuth-token，安全性考量
        if (!sameConnection(userResponse, url)) {
          requestBuilder.removeHeader("Authorization");
        }
        // 完活返回
        return requestBuilder.url(url).build();

      case HTTP_CLIENT_TIMEOUT:
      case HTTP_UNAVAILABLE:
        //  如果客户端超时，会采用各种方式重试，重试失败会返回null
        ...
        return userResponse.request();
        ...
        return null;

      default:
        return null;
    }
  }

  private int retryAfter(Response userResponse, int defaultDelay) {
    String header = userResponse.header("Retry-After");

    if (header == null) {
      return defaultDelay;
    }

    // https://tools.ietf.org/html/rfc7231#section-7.1.3
    // currently ignores a HTTP-date, and assumes any non int 0 is a delay
    if (header.matches("\\d+")) {
      return Integer.valueOf(header);
    }

    return Integer.MAX_VALUE;
  }

  /**
   * Returns true if an HTTP request for {@code followUp} can reuse the connection used by this
   * engine.
   */
  private boolean sameConnection(Response response, HttpUrl followUp) {
    HttpUrl url = response.request().url();
    return url.host().equals(followUp.host())
        && url.port() == followUp.port()
        && url.scheme().equals(followUp.scheme());
  }
}
```

### 总结

`RetryAndFollowUpInterceptor`的职责：

- 提供了异常重试功能；
- 提供固定不超过20次重定向功能；
- 提供「取消请求」功能。

本来根据单一原则，不应该把三个功能耦合到一个拦截器，但是为什么JW大神还是如此来设计呢？

而且我还有个疑问就是，重试次数竟然是固定不超过20次，不能由我们自己来设置。

### 解决不能自定义重试次数的问题

#### 方式一：自定义重试拦截器

首先关闭OkHttp自带的重试机制

```java
OkHttpClient.Builder()
            .retryOnConnectionFailure(false)
            .build()
```

然后自定义一个重试拦截器，如下：

```java
public class RetryIntercepter implements Interceptor {

  public int maxRetry;//最大重试次数
  private int retryNum = 0;//假如设置为3次重试的话，则最大可能请求4次（默认1次+3次重试）

  public RetryIntercepter(int maxRetry) {
      this.maxRetry = maxRetry;
  }

  @Override
  public Response intercept(Chain chain) throws IOException {
      Request request = chain.request();
      System.out.println("retryNum=" + retryNum);
      Response response = chain.proceed(request);
      while (!response.isSuccessful() && retryNum < maxRetry) {
          retryNum++;
          System.out.println("retryNum=" + retryNum);
          response = chain.proceed(request);
      }
      return response;
  }
}
```

>上面是我从网上找到的例子，其实他写的并不好，但是可以作为一个思路。

#### 方式二 RxJava的retryWhen操作符