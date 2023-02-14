---
title: GraphQL-Java(零)从SpringBoot服务端开始
date: 2021-01-23 16:38:19
tags: [GraphQL]
---

## 引入依赖

```groovy
dependencies {
    implementation 'com.graphql-java:graphql-java:14.1' // NEW
    implementation 'com.graphql-java:graphql-java-spring-boot-starter-webmvc:1.0' // NEW
    implementation 'com.google.guava:guava:26.0-jre' // NEW
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

## 定义Schema

在`src/main/resources`中创建`schema.graphqls`如下：

```sdl
type Query {
  bookById(id: ID): Book 
}

type Book {
  id: ID
  name: String
  pageCount: Int
  author: Author
}

type Author {
  id: ID
  firstName: String
  lastName: String
}
```

这里定义了一个顶级字段：`bookById`，它用来返回特定ID的book；

并且定义了一个Boo类，其中包含`id`,`name`,`pageCount`,`author`。`author`是另一个类型。

## 解析Schema

在项目中创建一个`GraphQLProvider`类型的java文件，用来创建GraphQL实例：

```java
@Component
public class GraphQLProvider {

    private GraphQL graphQL;

    @Bean
    public GraphQL graphQL() { 
        return graphQL;
    }

    /**
     * 用Guava Resources读取资源文件
     * 
     * @throws IOException
     */
    @PostConstruct
    public void init() throws IOException {
        URL url = Resources.getResource("schema.graphqls");
        String sdl = Resources.toString(url, Charsets.UTF_8);
        GraphQLSchema graphQLSchema = buildSchema(sdl);
        this.graphQL = GraphQL.newGraphQL(graphQLSchema).build();
    }

    private GraphQLSchema buildSchema(String sdl) {
      // TODO: we will create the schema here later 
    }
}
```

我们通过google的Guava `Resources`来读取Schema.graphql文件，并且通过`@Bean`注解把GraphQL实例暴露出去，GraphQL Java Spring adapter会使用这个GraphQL实例来把schema匹配到`/graphql'路径，以后通过这个路径调用所有接口。

接下来实现buildSchema方法：

```java
@Autowired
GraphQLDataFetchers graphQLDataFetchers;


/**
* 构建调用模型
* @param sdl 从资源文件获取到的sdl语言内容
* @return
*/
private GraphQLSchema buildSchema(String sdl) {
    TypeDefinitionRegistry typeRegistry = new SchemaParser().parse(sdl);
    RuntimeWiring runtimeWiring = buildWiring();
    SchemaGenerator schemaGenerator = new SchemaGenerator();
    return schemaGenerator.makeExecutableSchema(typeRegistry, runtimeWiring);
}

private RuntimeWiring buildWiring() {
    return RuntimeWiring.newRuntimeWiring()
            .type(newTypeWiring("Query")
                    .dataFetcher("bookById", graphQLDataFetchers.getBookByIdDataFetcher()))
            .type(newTypeWiring("Book")
                    .dataFetcher("author", graphQLDataFetchers.getAuthorDataFetcher()))
            .build();
}
```

`TypeDefinitionRegistry` 是SDL转换成的java对象。`RuntimeWiring`才是真正的重点，主要作用是注册了两个`DataFetchers`：

- 一个用来根据ID查book；
- 一个用来获取特定book的作者。

仔细看代码会发现，在构建`RuntimeWiring`时用到的字符串参数，全都是我们在SDL中定义好的type，所以这里的作用是告诉GraphQL Adapter怎么去根据SDL拿对应的数据。

## DataFetchers

DataFetcher是一个接口，用来根据SDL中的field来获取对应的数据，所以它内部只有一个抽象方法：

```java
public interface DataFetcher<T> {
    T get(DataFetchingEnvironment dataFetchingEnvironment) throws Exception;
}
```

这里要注意，SDL中每一个field都对应一个`DataFetcher`，如果没有特别指定一个`DataFetcher`，默认会采用`PropertyDataFetcher`。所以在上面生成`RuntimeWiring`的代码中，并没有对author注册dataFetcher，而是让他直接采用了默认的`PropertyDataFetcher`。

接下来我们就来实现`DataFetcher`了，创建一个`GraphQLDataFetchers`类：

```java
@Component
public class GraphQLDataFetchers {

    private static List<Map<String, String>> books = Arrays.asList(
            ImmutableMap.of("id", "book-1",
                    "name", "Harry Potter and the Philosopher's Stone",
                    "pageCount", "223",
                    "authorId", "author-1"),
            ImmutableMap.of("id", "book-2",
                    "name", "Moby Dick",
                    "pageCount", "635",
                    "authorId", "author-2"),
            ImmutableMap.of("id", "book-3",
                    "name", "Interview with the vampire",
                    "pageCount", "371",
                    "authorId", "author-3")
    );

    private static List<Map<String, String>> authors = Arrays.asList(
            ImmutableMap.of("id", "author-1",
                    "firstName", "Joanne",
                    "lastName", "Rowling"),
            ImmutableMap.of("id", "author-2",
                    "firstName", "Herman",
                    "lastName", "Melville"),
            ImmutableMap.of("id", "author-3",
                    "firstName", "Anne",
                    "lastName", "Rice")
    );

    public DataFetcher getBookByIdDataFetcher() {
        return dataFetchingEnvironment -> {
            String bookId = dataFetchingEnvironment.getArgument("id");
            return books
                    .stream()
                    .filter(book -> book.get("id").equals(bookId))
                    .findFirst()
                    .orElse(null);
        };
    }

    public DataFetcher getAuthorDataFetcher() {
        return dataFetchingEnvironment -> {
            Map<String,String> book = dataFetchingEnvironment.getSource();
            String authorId = book.get("authorId");
            return authors
                    .stream()
                    .filter(author -> author.get("id").equals(authorId))
                    .findFirst()
                    .orElse(null);
        };
    }
}
```

全都是mock数据，亮点是采用了java的新特性，通过流式调用来处理集合，并且还有lamda表达式的书写。

上面的代码中，我们可以看到，`DataFetcher`是`DataFetchingEnvironment`，三个查询方法也都是返回`DataFetchingEnvironment`的实例，其中需要关注的是：

- `getArgument("id")`用来获取参数。
- `getSource()`用来获取对象。

根据我们在SDL中的定义：

```sdl
type Query {
  bookById(id: ID): Book 
}
```

查询book时需要传递一个id，所以这个id怎么获取呢？通过`dataFetchingEnvironment.getArgument("id")`就能获取到。

然后根据SDL中定义：

```sdl
type Book {
  id: ID!
  name: String
  pageCount: Int
  author: Author
}
```

查询特定book的作者，那么需要一个特定的book实例，通过`getSource()`可以获取到父级的book实例。

在GraphQL中，**每一个field的DataFetcher是从上到下，从父到子被调用的，并且父级的结果可以从子级的`DataFetcherEnvironment`的`getSource()`方法中取到**。

### Default DataFetchers

前面说过，如果不给field指定一个DataFetcher，那么将默认使用`PropertyDataFetcher`。在这个demo里，意味着`Book.id`, `Book.name`, `Book.pageCount`, `Author.id`, `Author.firstName`, `Author.lastName`都对应着一个`PropertyDataFetcher`。

`PropertyDataFetcher`会通过多种方式从Java对象中找到对应的属性，如果是`java.util.Map`，就通过Key去找。这里要求field的名字与key一致。如果不一致就会返回null。（这里我翻译的不太对味，但是意思还是很明确的）。

## Try API

直接rest_client了：

```
POST http://localhost:9000/graphql
Content-Type: application/json
X-REQUEST-TYPE: GraphQL

{
    bookById(id:"book-1"){
        id
        name
        pageCount
        author {
            firstName
            lastName
        }
    }
}
```

返回结果：

```
HTTP/1.1 200 
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Sat, 23 Jan 2021 13:35:24 GMT
Connection: close

{
  "data": {
    "bookById": {
      "id": "book-1",
      "name": "Harry Potter and the Philosopher's Stone",
      "pageCount": 223,
      "author": {
        "firstName": "Joanne",
        "lastName": "Rowling"
      }
    }
  }
}
```