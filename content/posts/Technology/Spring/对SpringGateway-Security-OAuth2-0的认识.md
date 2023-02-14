---
title: 对SpringGateway+Security+OAuth2.0的认识
date: 2021-01-12 17:49:10
tags: [spring,cloud,security,oauth2.0]
---

趁着下载Keycloak的时候，记录一下自己对SpringCloud中的用户鉴权系统的认识。

<!-- more -->

## OAuth2.0 四种授权模式

|模式|refresh_token|用途|
|----|----|----|
|authorization_code|true|允许用户通过第三方应用登录自身获取资源，前提是用户已登录自身服务。|
|implicit|false|简化模式，跳过了获取授权码过程。|
|password|true|账号密码模式，用于高度授信场景，比如登录微信、QQ自身｜
|client_credentials|false|客户端模式，用于高度授信的其他服务，如企业自己的其他服务，或高度加密的硬件客户端，登录过程完全不需要用户操作。|


## Security、ResourceServer、ClientServer各自的作用和关系

* SpringSecurity:鉴权方式的配置，比如哪些接口需要鉴权，还要排除调登录注册和登出接口等等。还有用户信息的存放、认证，Token Provider 的配置等等。

* ResourceServer:是指需要收到保护的资源。比我的一个服务有大量接口，需要采用Security保护，那这个服务就是个ResourceServer。怎么保护呢？通过集成SpringSecurity来保护。每当有请求要访问我们的接口，ResourceServer都需要向token_provider验证token的有效性。

* ClientServer:是用来在服务端获取access_token和refresh_token用的，并且可以自动使用refresh_token去刷新access_token。向谁获取？可以是微信、微博、github等等。

* TokenProvider:用来提供token的服务。比如微信、支付宝、github,也可以是自己搭建的Keycloak服务。

## 网关统一用户认证

很简单，说白了就是在Gateway集成Security，这样以来Gateway就成了一个ResourceServer，并且可以为所有路由做用户认证。

但是！**具体的鉴权还是需要各个服务自己去做**，毕竟网关不知道具体的服务需要具体哪一项权限。

## FAQ

### 直接使用微信、支付宝的token来做自己的有效性验证可以码？

当然可以！只不过，人家的token验证只有在调用他们的服务时才会生效，如果不调用他们的服务，我们自己不知道人家的token是否有效（超时过期）。

那么该怎么实现？当我们自己的接口被访问时，先去调个人家的服务，最好是无关痛痒的服务，单纯为了验证人家的token是否有效。

但是这样以来就大大降低了接口的请求速度，而且也不优雅，平衡下来还不许自己搭建帐号体系来的实在！

### 采用授权码方式让自己的客户端访问自己服务？

这就叫脱了裤子放屁，为安全而安全！

用微信的第三方登录举例，假如我们没有登录微信，当使用微信时，依然需要用帐号密码方式先登录微信。因为你在操作授权登录时，是先调起微信的页面或者它SDK的页面，是在人家的页面里面玩的，对于微信来讲，他自己的页面就是高度授信的，所以追本溯源，授权码登录方式还是基于帐号密码之上的。