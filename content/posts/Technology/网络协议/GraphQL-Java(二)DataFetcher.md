---
title: GraphQL-Java(二)DataFetching
date: 2021-01-23 22:28:28
tags: [GraphQL]
---

## GraphQL如何获取数据

每个field都有一个`graphql.schema.DataFetcher与之对应`。

一些File会使用特定的data fetcher，这种DataFetcher知道怎么去访问数据库获取field的信息，然后大多数的dataFetcher则是简单的从内存中的object获取数据。在获取数据时，会参照field的名字and Plain Old Java Object (POJO) patterns to get the data.

所以可以像这样来声明一个type:

```sdl
type Query {
    products(match : String) : [Product]   # a list of products
}

type Product {
    id : ID
    name : String
    description : String
    cost : Float
    tax : Float
    launchDate(dateFormat : String = "dd, MMM, yyyy') : String
}
```

`Query.products`这个field有一个data fetcher，用来获取从数据库中获取`Product`类型的fields集合。它带有一个`match`参数，可以过滤出我们想要的product：

```java
DataFetcher productsDataFetcher = new DataFetcher<List<ProductDTO>>() {
    @Override
    public List<ProductDTO> get(DataFetchingEnvironment environment) {
        DatabaseSecurityCtx ctx = environment.getContext();

        List<ProductDTO> products;
        String match = environment.getArgument("match");
        if (match != null) {
            products = fetchProductsFromDatabaseWithMatching(ctx, match);
        } else {
            products = fetchAllProductsFromDatabase(ctx);
        }
        return products;
    }
};
```

每个`DataFetcher`相互之间传递着一个`graphql.schema.DataFetchingEnvironment`对象，这个对象包含正在获取的field、传递给field的参数arguments、和其他信息诸如field的类型，它父级的类型query root object或者query context object。

要注意的是，data fetcher的编码是基于context object作为一个安全程序来处理数据库的访问。这是一个基本技术来提供一个底层调用上下文。

当我们有一个`ProductDTO`的列表，我们不需要为每一个field都搞一个data fetcher。graphql-java 有一个聪明的`graphql.schema.PropertyDataFetcher`知道怎么去根据名称追踪POJO的结构。在本例中就是有个带有name的field，然后`graphql.schema.PropertyDataFetcher`将尝试通过POJO的`public String getName()`方法来获取数据。

`graphql.schema.PropertyDataFetcher`是默认的DataFetcher。

你仍然可以在DTO的方法中使用`graphql.schema.PropertyDataFetcher`来访问。这允许你在数据发送出去之前先做修改。比如，我们有个`launchDate`字段携带者一个可选的`dateFormat`参数。我们可以在ProductDTO中写个逻辑做日期时间的格式化。

```java
class ProductDTO {

    private ID id;
    private String name;
    private String description;
    private Double cost;
    private Double tax;
    private LocalDateTime launchDate;

    // ...

    public String getName() {
        return name;
    }

    // ...

    public String getLaunchDate(DataFetchingEnvironment environment) {
        String dateFormat = environment.getArgument("dateFormat");
        return yodaTimeFormatter(launchDate,dateFormat);
    }
}
```

## 定制PropertyDataFetcher

像之前讲的一样，`graphql.schema.PropertyDataFetcher`是默认的data fetcher，并且它在获取数据时使用标准结构。

它支持POJO和map。假设有个field叫`fieldX`，它将会寻找一个POJO也叫做`fieldX`，或者在Map中去找一个key也叫做`fieldX`。

然而，你在定义的schema与POJO有一点不通，比如`Product.description`对应着POJO中的`getDesc()`.

你可以在SDL中使用`@fetch`注解来做匹配：

```sdl
directive @fetch(from : String!) on FIELD_DEFINITION

type Product {
    id : ID
    name : String
    description : String @fetch(from:"desc")
    cost : Float
    tax : Float
}
```

这将告诉`graphql.schema.PropertyDataFetcher`在获取数据时应该使用名为`desc`的属性来对应SDL中的`description`字段。

如果你不用SDL,而是喜欢手写代码，那么可以直接指定Data Fetcher:

```java
GraphQLFieldDefinition descriptionField = GraphQLFieldDefinition.newFieldDefinition()
                .name("description")
                .type(Scalars.GraphQLString)
                .build();

        GraphQLCodeRegistry codeRegistry = GraphQLCodeRegistry.newCodeRegistry()
                .dataFetcher(
                        coordinates("ObjectType", "description"),
                        PropertyDataFetcher.fetching("desc"))
                .build();
```

## DataFetchingEnvironment中有意思的部分

每一个data fetcher都传递了一个`graphql.schema.DataFetchingEnvironment`对象。它知道我们正在获取什么东西，并且知道提供了哪些参数。

- `<T> T getSource()`-获取field的信息。这个对象就是父级获取到的结果，他一个内存中的DTO对象，并且通过POJO的getters获取到值。在复杂的场景中，你可以通过检查它来获取field的特定信息。当graphql的field tree执行后返回的每一个field的值，都会成为子级的`source`。
- `<T> T getRoot()`-用来查看graphql的查询query。在顶层级中，root和source是一样的。root object在查询期间是不会改变的，它可能为null，所以尽量不要用这玩意。
- `Map<String, Object> getArguments()`-用来获取field提供的参数。
- `<T> T getContext()`-当query第一次执行时，才会设置好context，并让其停留在query的生命周期上。所以context的生命周期与query一致。context可以是任何值，主要是data fetcher在获取数据时需要这玩意。所以只需要把它注入给data fetcher就行了，其他场景都用不到。（日了狗了，讲了这么长一串，最后特么没点鸟用）
- `ExecutionStepInfo getExecutionStepInfo()`-用来获取`ExecutionStepInfo`。当query执行后，`ExecutionStepInfo`存放了所有field的类型信息。
- `DataFetchingFieldSelectionSet getSelectionSet()`-用来获取选项集。所谓「选项集」就是当前执行中的field之后的哪些被「选中」的子filed们。这对于展望接下来将要执行的field的信息。（我感觉就像是音乐播放器在播放收藏的专辑一样，比如当前播放到了费玉清的《天之大》，那么我们就可以通过这个方法去查看《天之大》下面有哪些歌将要被播放了）。
- `ExecutionId getExecutionId()`-所有的查询都有一个unique id。可以把id作为log日志的tag，用来打印特定的query。

### ExecutionStepInfo好玩的地方

一个graphql query在执行时会生成一个call tree，就是「调用树」。`graphql.execution.ExecutionStepInfo.getParentTypeInfo`允许你向上导航，看到是什么类型的field被带到了当前的执行过程。

在整个执行期间，就形成了一个树形path，`graphql.execution.ExecutionStepInfo.getPath`方法可以返回这个树形path的描述。这在日志打印和调试query时比较有用。而且可以查看到非空类型的名称和折叠后的列表。

一句话总结就是，主要用于调试和排错。

### DataFetchingFieldSelectionSet有趣的地方

想象一下有个query就像这样：

```sdl
query {
    products {
        # the fields below represent the selection set
        name
        description
        sellingLocations {
            state
        }
    }
}
```

`Product`的所有子field，对于`Product`来讲就是`selection fields`。当我们想要知道哪些子级field正在被请求时非常有用。举例来讲，我们的POJO中可能有很多属性，数据库表中的字段也与之一一对应。但是我们跑个graphql查询时可能不需要返回全部的字段，我们只要我们需要的字段就行了，此时data fetcher去查数据库时也不需要查出所有的列，也是按需获取就行了。那怎么在代码里感知到我们具体请求的哪些字段呢？就是靠这个`DataFetchingFieldSelectionSet`。

比如上例中我们请求了`sellingLocations`字段，或者我们使用更高效的查询：同时查出`products`和`sellingLocations`。