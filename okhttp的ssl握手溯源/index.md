# OkHttp的SSL握手溯源

最近研究PKI，想实现私钥不出TEE这个需求,需要确认okhttp中SSL认证的实现，结果说看看源码吧，让我好一顿找阿，特此记录一下过程。
<!--more-->

Okhttp添加自签名证书方法:

```java
OkHttpClient.Builder builder = new OkHttpClient.Builder();

// 添加自定义的SSLSocketFactory和X509TrustManager
builder.sslSocketFactory(sslInput.mSSLSocketFactory, sslInput.mX509TrustManager);

// 域名验证
builder.hostnameVerifier((hostname, session) ->
        true
);
```

最主要的其实就是`builder.sslSocketFactory`函数，由于是建造者模式，那么OkhttpClient.Builder设置了属性，最终在调用`builder.build()`函数时，一定会把builder的属性赋值给父类`OkhttpClient`，追踪`builder.build()`函数，最终调用`OkhttpClient`的构造函数，其中有一段逻辑：

```java
if (builder.sslSocketFactory != null || !isTLS) {
    //如果builder设置了sslSocketFactory就用builder的
    this.sslSocketFactory = builder.sslSocketFactory;
    this.certificateChainCleaner = builder.certificateChainCleaner;
} else {
    // 如果没有设置过builder的ssSocketFactory就用系统默认的
    X509TrustManager trustManager = systemDefaultTrustManager();
    this.sslSocketFactory = systemDefaultSslSocketFactory(trustManager);
    this.certificateChainCleaner = CertificateChainCleaner.get(trustManager);
}
```

我们主要关注的就是系统默认的`systemDefaultSslSocketFactory(trustManager)`。

继续追踪：

```java
private SSLSocketFactory systemDefaultSslSocketFactory(X509TrustManager trustManager) {
    try {
      // 获取到平台的SSLContext
      SSLContext sslContext = Platform.get().getSSLContext();
      sslContext.init(null, new TrustManager[] { trustManager }, null);
      return sslContext.getSocketFactory();
    } catch (GeneralSecurityException e) {
      throw assertionError("No System TLS", e); // The system has no TLS. Just give up.
    }
  }
```

这里就看到了Okhttp的跨平台思路，简单看一下`Platform.get()`的逻辑：

```java
public class Platform {

  // 静态加载方式
  private static final Platform PLATFORM = findPlatform();

  public static Platform get() {
    return PLATFORM;
  }

  private static Platform findPlatform() {

    // 优先加载安卓平台
    Platform android = AndroidPlatform.buildIfSupported();

    if (android != null) {
      return android;
    }

    // 其他的都是jdk中的安全策略
    ...
  }
}
```

>那么从这里的逻辑可以看到，各个平台的安全策略可能是不一致的。

那么OKHttp是怎么加载安卓平台的呢：

```java
public static Platform buildIfSupported() {
    // Attempt to find Android 2.3+ APIs.
    try {
      Class<?> sslParametersClass;
      try {
        sslParametersClass = Class.forName("com.android.org.conscrypt.SSLParametersImpl");
      } catch (ClassNotFoundException e) {
        // Older platform before being unbundled.
        sslParametersClass = Class.forName(
            "org.apache.harmony.xnet.provider.jsse.SSLParametersImpl");
      }

      ...

      return new AndroidPlatform(sslParametersClass, setUseSessionTickets, setHostname,
          getAlpnSelectedProtocol, setAlpnProtocols);
    } catch (ClassNotFoundException ignored) {
      // This isn't an Android runtime.
    }

    return null;
  }
```

主要是通过hook系统中的`com.android.org.conscrypt.SSLParametersImpl`。可以先记住这个类，暂时跳过这里。

## SSLContext

我们继续来看`SSLContext sslContext = Platform.get().getSSLContext()`，向里面追踪：

```java

import sun.security.jca.GetInstance;

public SSLContext getSSLContext() {
  try {
    return SSLContext.getInstance("TLS");
  } catch (NoSuchAlgorithmException e) {
    throw new IllegalStateException("No TLS provider", e);
  }
}

public static SSLContext getInstance(String protocol)
            throws NoSuchAlgorithmException {
    GetInstance.Instance instance = GetInstance.getInstance
            ("SSLContext", SSLContextSpi.class, protocol);
    return new SSLContext((SSLContextSpi)instance.impl, instance.provider,
            protocol);
}
```

发现GetInstance.Instance.impl是`SSLContextSpi`类型。

到这里`GetInstance`的源码无法查看，是因为我们没有下载`sun.security.jca.GetInstance`的源码，但是在AOSP中是可以搜索到的,[文件在此](GetInstance.java)。

核心方法：

```java
/**
 * @param type = "SSLContext"
 * @param clazz = SSLContextSpi.class
 * @param algorithm = "TLS"
 **/
public static Instance getInstance(String type, Class<?> clazz,
        String algorithm) throws NoSuchAlgorithmException {

    ProviderList list = Providers.getProviderList();
    Service firstService = list.getService(type, algorithm);
    if (firstService == null) {
        throw new NoSuchAlgorithmException
                (algorithm + " " + type + " not available");
    }
    NoSuchAlgorithmException failure;
    try {
        return getInstance(firstService, clazz);
    } catch (NoSuchAlgorithmException e) {
        failure = e;
    }
    
    for (Service s : list.getServices(type, algorithm)) {
        if (s == firstService) {
            // do not retry initial failed service
            continue;
        }
        try {
            return getInstance(s, clazz);
        } catch (NoSuchAlgorithmException e) {
            failure = e;
        }
    }
    throw failure;
}
```

这里主要是用参数中的type与algorithm去做个命中匹配，具体是通过`Providers`的`Providers.getProviderList()`方法完成,而这个类同样位于`sun.security.jca`包中，在AOSP内是可见的：

```java
public static ProviderList getProviderList() {
    ProviderList list = getThreadProviderList();
    if (list == null) {
        list = getSystemProviderList();
    }
    return list;
}

private static void setSystemProviderList(ProviderList list) {
    providerList = list;
}
```

可惜的是，AOSP中并没有找到调用`setSystemProviderList()`函数的地方，那就有可能是外部hook调用了。

### 接下来关注`Service firstService = list.getService(type, algorithm);`

```java
public List<Service> getServices(String type, String algorithm) {
    return new ServiceList(type, algorithm);
}
```



.....

## 最终

是在`external/conscrypt/common/src/main/java/org/conscrypt/NativeSsl.java`中有具体的握手逻辑，最终是调用：

```java
NativeCrypto.SSL_do_handshake(ssl, this, fd, handshakeCallbacks, timeoutMillis);
```

而这个是个Native方法，具体实现是在`external/conscrypt/common/src/jni/main/cpp/conscrypt/native_crypto.cc`中实现。

```c
/**
 * Perform SSL handshake
 */
static void NativeCrypto_SSL_do_handshake(JNIEnv* env, jclass, jlong ssl_address, CONSCRYPT_UNUSED jobject ssl_holder, jobject fdObject,
                                          jobject shc, jint timeout_millis) {
    CHECK_ERROR_QUEUE_ON_RETURN;
    SSL* ssl = to_SSL(env, ssl_address, true);
    JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake fd=%p shc=%p timeout_millis=%d", ssl, fdObject,
              shc, timeout_millis);
    if (ssl == nullptr) {
        return;
    }
    if (fdObject == nullptr) {
        conscrypt::jniutil::throwNullPointerException(env, "fd == null");
        JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake fd == null => exception", ssl);
        return;
    }
    if (shc == nullptr) {
        conscrypt::jniutil::throwNullPointerException(env, "sslHandshakeCallbacks == null");
        JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake sslHandshakeCallbacks == null => exception",
                  ssl);
        return;
    }

    NetFd fd(env, fdObject);
    if (fd.isClosed()) {
        // SocketException thrown by NetFd.isClosed
        JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake fd.isClosed() => exception", ssl);
        return;
    }

    int ret = SSL_set_fd(ssl, fd.get());
    JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake s=%d", ssl, fd.get());

    if (ret != 1) {
        conscrypt::jniutil::throwSSLExceptionWithSslErrors(env, ssl, SSL_ERROR_NONE,
                                                           "Error setting the file descriptor");
        JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake SSL_set_fd => exception", ssl);
        return;
    }

    /*
     * Make socket non-blocking, so SSL_connect SSL_read() and SSL_write() don't hang
     * forever and we can use select() to find out if the socket is ready.
     */
    if (!conscrypt::netutil::setBlocking(fd.get(), false)) {
        conscrypt::jniutil::throwSSLExceptionStr(env, "Unable to make socket non blocking");
        JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake setBlocking => exception", ssl);
        return;
    }

    AppData* appData = toAppData(ssl);
    if (appData == nullptr) {
        conscrypt::jniutil::throwSSLExceptionStr(env, "Unable to retrieve application data");
        JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake appData => exception", ssl);
        return;
    }

    ret = 0;
    SslError sslError;
    while (appData->aliveAndKicking) {
        errno = 0;

        if (!appData->setCallbackState(env, shc, fdObject)) {
            // SocketException thrown by NetFd.isClosed
            JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake setCallbackState => exception", ssl);
            return;
        }
        ret = SSL_do_handshake(ssl);
        appData->clearCallbackState();
        // cert_verify_callback threw exception
        if (env->ExceptionCheck()) {
            ERR_clear_error();
            JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake exception => exception", ssl);
            return;
        }
        // success case
        if (ret == 1) {
            break;
        }
        // retry case
        if (errno == EINTR) {
            continue;
        }
        // error case
        sslError.reset(ssl, ret);
        JNI_TRACE(
                "ssl=%p NativeCrypto_SSL_do_handshake ret=%d errno=%d sslError=%d "
                "timeout_millis=%d",
                ssl, ret, errno, sslError.get(), timeout_millis);

        /*
         * If SSL_do_handshake doesn't succeed due to the socket being
         * either unreadable or unwritable, we use sslSelect to
         * wait for it to become ready. If that doesn't happen
         * before the specified timeout or an error occurs, we
         * cancel the handshake. Otherwise we try the SSL_connect
         * again.
         */
        if (sslError.get() == SSL_ERROR_WANT_READ || sslError.get() == SSL_ERROR_WANT_WRITE) {
            appData->waitingThreads++;
            int selectResult = sslSelect(env, sslError.get(), fdObject, appData, timeout_millis);

            if (selectResult == THROWN_EXCEPTION) {
                // SocketException thrown by NetFd.isClosed
                JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake sslSelect => exception", ssl);
                return;
            }
            if (selectResult == -1) {
                conscrypt::jniutil::throwSSLExceptionWithSslErrors(
                        env, ssl, SSL_ERROR_SYSCALL, "handshake error",
                        conscrypt::jniutil::throwSSLHandshakeExceptionStr);
                JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake selectResult == -1 => exception",
                          ssl);
                return;
            }
            if (selectResult == 0) {
                conscrypt::jniutil::throwSocketTimeoutException(env, "SSL handshake timed out");
                ERR_clear_error();
                JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake selectResult == 0 => exception",
                          ssl);
                return;
            }
        } else {
            // ALOGE("Unknown error %d during handshake", error);
            break;
        }
    }

    // clean error. See SSL_do_handshake(3SSL) man page.
    if (ret == 0) {
        /*
         * The other side closed the socket before the handshake could be
         * completed, but everything is within the bounds of the TLS protocol.
         * We still might want to find out the real reason of the failure.
         */
        if (sslError.get() == SSL_ERROR_NONE ||
            (sslError.get() == SSL_ERROR_SYSCALL && errno == 0) ||
            (sslError.get() == SSL_ERROR_ZERO_RETURN)) {
            conscrypt::jniutil::throwSSLHandshakeExceptionStr(env, "Connection closed by peer");
        } else {
            conscrypt::jniutil::throwSSLExceptionWithSslErrors(
                    env, ssl, sslError.release(), "SSL handshake terminated",
                    conscrypt::jniutil::throwSSLHandshakeExceptionStr);
        }
        JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake clean error => exception", ssl);
        return;
    }

    // unclean error. See SSL_do_handshake(3SSL) man page.
    if (ret < 0) {
        /*
         * Translate the error and throw exception. We are sure it is an error
         * at this point.
         */
        conscrypt::jniutil::throwSSLExceptionWithSslErrors(
                env, ssl, sslError.release(), "SSL handshake aborted",
                conscrypt::jniutil::throwSSLHandshakeExceptionStr);
        JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake unclean error => exception", ssl);
        return;
    }
    JNI_TRACE("ssl=%p NativeCrypto_SSL_do_handshake => success", ssl);
}
```
