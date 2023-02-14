---
title: "BridgeInterceptor"
subtitle: "桥接拦截器"
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
...
interceptors.add(new BridgeInterceptor(client.cookieJar()));
...
```

RealCall在创建这个拦截器的时候只传递了一个`cookieJar`。

### 作用

这个类的注释如下：

>Bridges from application code to network code. First it builds a network request from a user request. Then it proceeds to call the network. Finally it builds a user response from the network response.

意思是，在应用程序的code与网络连接的code之间桥接。先通过用户的request创建一个网络request。然后执行网络任务。最后从网络的response创建一个用户的响应response。

>翻译完了类注释，再来看代码就很好理解了。

### BridgeInterceptor的处理过程

```java
public final class BridgeInterceptor implements Interceptor {
  
  private final CookieJar cookieJar;

  public BridgeInterceptor(CookieJar cookieJar) {
    this.cookieJar = cookieJar;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    
    //------------------request处理

    // 从「拦截器管理器」中拿到所谓的用户request
    Request userRequest = chain.request();

    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();

    // 各种桥接转换
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());

    // 附加cookie
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    // 附加OkHttp自己的UA
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    //------------------response处理

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }

  /** Returns a 'Cookie' HTTP request header with all cookies, like {@code a=b; c=d}. */
  private String cookieHeader(List<Cookie> cookies) {
    StringBuilder cookieHeader = new StringBuilder();
    for (int i = 0, size = cookies.size(); i < size; i++) {
      if (i > 0) {
        cookieHeader.append("; ");
      }
      Cookie cookie = cookies.get(i);
      cookieHeader.append(cookie.name()).append('=').append(cookie.value());
    }
    return cookieHeader.toString();
  }
}
```

### 总结

看完之后，我的感想就是**一定不要相信类注释**。

这个拦截器确实是类注释描述的那样，把用户的请求转为网络请求，把网络响应转为用户响应，但是：
  - 处理request过程中，附加了cookie和自己的UA；
  - 处理response过程中，为response附加了请求中的url，以便于辨认。