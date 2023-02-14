---
title: "Quarkus读取配置"
date: 2021-09-07T17:45:01+08:00
lastmod: 2021-09-07T17:45:01+08:00
draft: false
authors: []
description: ""

tags: []
categories: []
series: [Quarkus]

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

quarkus可以从.properties文件中读取配置。

<!--more-->

## 定义配置

在.properties文件中定义配置：

```
greeting.message = hello
greeting.name = quarkus
```


### 配置源

1. (400) System properties
2. (300) Environment variables
3. (295) 当前工作目录中的.env文件
4. (260) Quarkus Application configuration file in $PWD/config/application.properties
5. (250) Quarkus Application configuration file application.properties in classpath
6. (100) MicroProfile Config configuration file META-INF/microprofile-config.properties in classpath

最终的配置是上面这些源中配置的聚合，quarkus是从低序号到高序号进行merge，序号越高就越靠后加载，包证了系统属性的安全性，也意味着我们可以在高序号源中定义相同的配置来覆盖低序号源中的配置。

#### System properties

系统属性可以在启动期间将配置传递给应用程序，比如：

```
./mvnw quarkus:dev -Dquarkus.datasource.password=youshallnotpass

or

java -Dquarkus.datasource.password=youshallnotpass -jar target/quarkus-app/quarkus-run.jar
```

#### Environment variables

就是系统的环境变量。比如：

```
export QUARKUS_DATASOURCE_PASSWORD=youshallnotpass ; java -jar target/quarkus-app/quarkus-run.jar
```

#### .env文件

```
QUARKUS_DATASOURCE_PASSWORD=youshallnotpass
```

一般位于根目录中，并且不推荐加入VersionControl中。

#### ApplicationConfiguration文件

##### 文件位置

- 一般位于`src/main/resources/application.properties`中
- 在测试系统中也可以有单独的配置文件`src/test/resources/application.properties`
- 也可以加载jar中的配置


##### 文件内容：

```
// 用户可以自定义配置
greeting.message=hello

// 系统扩展也从这里加载配置
quarkus.http.port=9090 
```

##### 多环境配置

quarkus默认有三个环境配置：

- dev 开发时激活（quarkus:dev）
- test 测试时激活
- prod 默认配置

格式：`%{profile-name}.config.name`

eg:

```
quarkus.http.port=9090
%dev.quarkus.http.port=8181
```

一般情况下，端口号是9090，但是当处于dev环境时则是8181。**如果dev环境没有设置值，则将使用默认值**。

##### 自定义环境配置

```
quarkus.http.port=9090
%custom.quarkus.http.port=9999
```

##### 父配置

可以通过`quarkus.config.profile.parent`设置一个父配置，如果在当前活跃的配置中找不到某个属性，就会去查找父配置：

```
quarkus.profile=dev
quarkus.config.profile.parent=common

%common.quarkus.http.port=9090
%dev.quarkus.http.ssl-port=9443

quarkus.http.port=8080
quarkus.http.ssl-port=8443
```

解释：当前采用dev环境，父配置是`common`，`quarkus.http.port`会先查找dev环境，发现dev没有设置这一项，再去查找父配置，所以最后实际上会采用父配置的值9090;`quarkus.http.ssl-port`先去查找dev环境，找到了值9443，就不会再去查找父配置，所以最终值是9443。

#### 属性表达式

语法：

- `${ …​ }` 一般使用
- `${expression:value}` 默认值方式
- `${my.prop${compose}}` 组合表达式
- `${my.prop}${my.prop}` 多表达式
- `${quarkus.uuid}` 生成UUID

eg:

```
remote.host=quarkus.io

// 可以直接使用上面定义好的属性
callable.url=https://${remote.host}/
```

#### 索引属性

```
my.collection=dog,cat,turtle

my.indexed.collection[0]=dog
my.indexed.collection[1]=cat
my.indexed.collection[2]=turtle
```

### 自带配置

[Quarkus自带的配置属性](https://quarkus.io/guides/all-config)，其中带锁的是构建时配置，无法在运行时修改。

## 读取配置信息

### 注入方式

可以在resource中注入配置信息：

```java
@Path("/greeting")
public class GreetingResource {

    @ConfigProperty(name = "greeting.message")
    String message;

    /**
     * 默认值
     */
    @ConfigProperty(name = "greeting.suffix", defaultValue = "!")
    String suffix;

    /**
     * 可选方式，可以避免空指针
     */
    @ConfigProperty(name = "greeting.name")
    Optional<String> name;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return message + " " + name.orElse("world") + suffix;
    }
}
```

### 获取方式

```java
@GET
@Produces(MediaType.TEXT_PLAIN)
public String hello() {
    String message = ConfigProvider.getConfig().getValue("greeting.message", String.class);
    Optional<String> name = ConfigProvider.getConfig().getOptionalValue("greeting.name", String.class);

    return message + " " + name.get();
}
```

### 装箱方式

假设有配置：

```
greeting.message = hello
greeting.name = quarkus
```

我们可以使用`@ConfigMapping`把所有同一前缀的属性装箱成一个接口：

```java
// 支持逗号分隔多个
@ConfigMapping(prefix = "greeting")
public interface Greeting {

    // 命名必须与属性一致
    String name();

    String message();

}
```

然后在调用时就可以直接注入实现类：

```java
@Path("/greeting")
public class GreetingResource {

    @Inject
    Greeting greeting;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() {
        return greeting.message() + greeting.name();
    }
}
```

#### 嵌套方式

```java
@ConfigMapping(prefix = "server")
public interface Server {
    String host();

    int port();

    Log log();

    interface Log {
        boolean enabled();

        String suffix();

        boolean rotate();
    }
}
```

#### 覆盖属性名

##### @WithName

当我们的方法命名与配置文件中的属性名称不一致时，可以使用`@WithName`注解标明：

配置：

```
server.name=localhost
server.port=8080
```

映射：

```java
@ConfigMapping(prefix = "server")
interface Server {
    @WithName("name")
    String host();

    int port();
}
```

##### @WithParent

组合方式。

```
server.host=localhost
server.port=8080
server.name=konoha
```

```java
interface Server {
    @WithParentName
    ServerHostAndPort hostAndPort();

    @WithParentName
    ServerInfo info();
}

interface ServerHostAndPort {
    String host();

    int port();
}

interface ServerInfo {
    String name();
}
```

##### 驼峰命名策略

```
server.the-host=localhost
server.the-port=8080
```

用-来分隔单词，然后就可以使用驼峰方式命名的方法来映射：

```java
@ConfigMapping(prefix = "server")
interface Server {
    String theHost();

    int thePort();
}
```

##### 映射方法

配置：

```
foo=foo
```

映射：

```java
@ConfigMapping
public interface Converters {
    @WithConverter(FooBarConverter.class)
    String foo();
}

public static class FooBarConverter implements Converter<String> {
    @Override
    public String convert(final String value) {
        return "bar";
    }
}
```

当我们使用 `foo()`方法时，它并不会输出foo，而是输出bar。

##### 映射集合

配置：

```
server.environments[0].name=dev
server.environments[0].apps[0].name=rest
server.environments[0].apps[0].services=bookstore,registration
server.environments[0].apps[0].databases=pg,h2
server.environments[0].apps[1].name=batch
server.environments[0].apps[1].services=stock,warehous
```

映射：

```java
@ConfigMapping(prefix = "server")
public interface ServerCollections {
    Set<Environment> environments();

    interface Environment {
        String name();

        List<App> apps();

        interface App {
            String name();

            List<String> services();

            Optional<List<String>> databases();
        }
    }
}
```

##### 映射Map

配置：

```
server.host=localhost
server.port=8080
server.form.login-page=login.html
server.form.error-page=error.html
server.form.landing-page=index.html
```

映射：

```java
@ConfigMapping(prefix = "server")
public interface Server {
    String host();

    int port();

    Map<String, String> form();
}
```

##### 默认值

```java
public interface Defaults {
    @WithDefault("foo")
    String foo();

    @WithDefault("bar")
    String bar();
}
```

##### 验证配置

```java
@ConfigMapping(prefix = "server")
interface Server {
    @Size(min = 2, max = 20)
    String host();

    @Max(10000)
    int port();
}
```

需要`quarkus-hibernate-validator`扩展。

## 扩展配置

一般情况下，我们在实际的工作中可能不止dev、test、prod三种环境，可能还需要更多的配置。

### 自定义配置源

#### 扩展ConfigSource

```java
package org.acme.config;

public class InMemoryConfigSource implements ConfigSource {

    /**
     * 内存缓存
     */
    private static final Map<String, String> configuration = new HashMap<>();

    static {
        configuration.put("my.prop", "1234");
    }

    /**
     * 定义加载序号
     */
    @Override
    public int getOrdinal() {
        return 275;
    }

    @Override
    public Set<String> getPropertyNames() {
        return configuration.keySet();
    }

    @Override
    public String getValue(final String propertyName) {
        return configuration.get(propertyName);
    }

    /**
     * 返回该配置的名称
     */
    @Override
    public String getName() {
        return InMemoryConfigSource.class.getSimpleName();
    }
}
```

#### 注册配置源

接下来在`META-INF/services/org.eclipse.microprofile.config.spi.ConfigSource`注册配置：

```
org.acme.config.InMemoryConfigSource
```

#### 初始化配置源

当 Quarkus 应用程序启动时，可以初始化两次。一次用于静态初始化，第二次用于运行时初始化

##### 静态初始化

