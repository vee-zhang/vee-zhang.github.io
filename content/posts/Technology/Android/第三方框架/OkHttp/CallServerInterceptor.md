---
title: "CallServerInterceptor"
subtitle: "呼叫服务拦截器"
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

<!--more-->

### CallServerInterceptor 

负责实际与server交互（写读），并且完成了报文的封装与解析。

```java
@Override public Response intercept(Chain chain) throws IOException {

  RealInterceptorChain realChain = (RealInterceptorChain) chain;

  // 用来编码http请求和解码http响应
  HttpCodec httpCodec = realChain.httpStream();

  // 此时应该为null
  StreamAllocation streamAllocation = realChain.streamAllocation();

  // 此时应该为null
  RealConnection connection = (RealConnection) realChain.connection();

  // 拿到request
  Request request = realChain.request();

  long sentRequestMillis = System.currentTimeMillis();

  // 调用状态监听器，并且传递AsyncCall对象
  realChain.eventListener().requestHeadersStart(realChain.call());

  // 执行写入
  httpCodec.writeRequestHeaders(request);

  // 通知状态
  realChain.eventListener().requestHeadersEnd(realChain.call(), request);

  Response.Builder responseBuilder = null;

  // 如果不是GET或HEAD请求，并且带有请求体
  if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
    // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
    // Continue" response before transmitting the request body. If we don't get that, return
    // what we did get (such as a 4xx response) without ever transmitting the request body.
    if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {

      // 写入request完成
      httpCodec.flushRequest();

      realChain.eventListener().responseHeadersStart(realChain.call());

      // 开始读取response
      responseBuilder = httpCodec.readResponseHeaders(true);
    }

    if (responseBuilder == null) {
      // Write the request body if the "Expect: 100-continue" expectation was met.
      realChain.eventListener().requestBodyStart(realChain.call());
      long contentLength = request.body().contentLength();
      CountingSink requestBodyOut =
          new CountingSink(httpCodec.createRequestBody(request, contentLength));
      BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

      request.body().writeTo(bufferedRequestBody);
      bufferedRequestBody.close();
      realChain.eventListener()
          .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
    } else if (!connection.isMultiplexed()) {
      // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
      // from being reused. Otherwise we're still obligated to transmit the request body to
      // leave the connection in a consistent state.
      streamAllocation.noNewStreams();
    }
  }

  httpCodec.finishRequest();

  if (responseBuilder == null) {
    realChain.eventListener().responseHeadersStart(realChain.call());
    responseBuilder = httpCodec.readResponseHeaders(false);
  }

  // 构建response
  Response response = responseBuilder
      .request(request)
      .handshake(streamAllocation.connection().handshake())
      .sentRequestAtMillis(sentRequestMillis)
      .receivedResponseAtMillis(System.currentTimeMillis())
      .build();

  int code = response.code();
  if (code == 100) {
    // server sent a 100-continue even though we did not request one.
    // try again to read the actual response
    // 100状态码，用来表示请求被正常接受，需进一步处理，需要客户端继续请求
    responseBuilder = httpCodec.readResponseHeaders(false);

    response = responseBuilder
            .request(request)
            .handshake(streamAllocation.connection().handshake())
            .sentRequestAtMillis(sentRequestMillis)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build();

    code = response.code();
  }

  realChain.eventListener()
          .responseHeadersEnd(realChain.call(), response);

  if (forWebSocket && code == 101) {
    // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
    // 101状态码表示服务器已收到请求，但是需要客户端切换协议来进一步处理
    response = response.newBuilder()
        .body(Util.EMPTY_RESPONSE)
        .build();
  } else {
    response = response.newBuilder()
        .body(httpCodec.openResponseBody(response))
        .build();
  }

  if ("close".equalsIgnoreCase(response.request().header("Connection"))
      || "close".equalsIgnoreCase(response.header("Connection"))) {
    streamAllocation.noNewStreams();
  }

  // 204/205状态码表示服务器已成功处理请求，但是不会返回响应体。
  if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
    throw new ProtocolException(
        "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
  }

  return response;
}
```