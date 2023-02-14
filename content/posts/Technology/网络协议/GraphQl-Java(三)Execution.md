---
title: GraphQl-Java(三)Execution
date: 2021-01-19 10:13:53
tags: [GraphQL]
description: 本文基于GraphQl-java:v1.6版本官方文档中文翻译。
---

## Queries

如果想要在schema里跑个查询，需要用对应的参数创建一个新的`GraphQL object`，然后执行`execute()`。该方法会返回一个`ExecutionResult`，它包含查询结果，或者一个错误列表。

```java
GraphQLSchema schema = GraphQLSchema.newSchema()
        .query(queryType)
        .build();

GraphQL graphQL = GraphQL.newGraphQL(schema)
        .build();

ExecutionInput executionInput = ExecutionInput.newExecutionInput().query("query { hero { name } }")
        .build();

ExecutionResult executionResult = graphQL.execute(executionInput);

Object data = executionResult.getData();
List<GraphQLError> errors = executionResult.getErrors();
```

更多的列子在[这里](https://github.com/graphql-java/graphql-java/blob/master/src/test/groovy/graphql/StarWarsQueryTest.groovy)

## Data Fetchers

每一个field都对应着一个`graphql.schema.DataFetcher`，也被称作**resolvers**。

你可以用基于`graphql.schema.PropertyDataFetcher`去测试Java POJO对象提供的field。如果你没有为field指定一个`data fetcher`，那么默认会使用`graphql.schema.PropertyDataFetcher`。

然而，你需要通过自定义的data fetcher去获取域的顶层对象，这意味着你可能需要创建一个数据库，或者通过http调用其他服务获取。

`graphql-java`并不关注如何获取对象，这些逻辑需要你自己去处理。

一个data fetcher可能像这样：

```java
DataFetcher userDataFetcher = new DataFetcher() {
    @Override
    public Object get(DataFetchingEnvironment environment) {
        return fetchUserFromDatabase(environment.getArgument("userId"));
        }
};
```

每一个`DataFetcher`实际上都包含在`graphql.schema.DataFetchingEnvironment`对象中，而`graphql.schema.DataFetchingEnvironment`对象同时包含了正在获取的字段、已提供给字段的参数、字段的父级对象、查询根对象或者查询上下文。

在上面的示例中，执行将等待数据提取程序返回后再继续。您可以通过向数据返回 CompletionStage 来异步执行 DataFetcher，本页面将对此进行更详细的解释。

## Exceptions while fetching data

如果在数据读取器调用期间发生异常，那么默认情况下执行策略将生成 graphql。然后将其添加到结果的错误列表中。记住，graphql 允许带有错误的部分结果。

```java

    public class SimpleDataFetcherExceptionHandler implements DataFetcherExceptionHandler {
        private static final Logger log = LoggerFactory.getLogger(SimpleDataFetcherExceptionHandler.class);

        @Override
        public void accept(DataFetcherExceptionHandlerParameters handlerParameters) {
            Throwable exception = handlerParameters.getException();
            SourceLocation sourceLocation = handlerParameters.getField().getSourceLocation();
            ExecutionPath path = handlerParameters.getPath();

            ExceptionWhileDataFetching error = new ExceptionWhileDataFetching(path, exception, sourceLocation);
            handlerParameters.getExecutionContext().addError(error);
            log.warn(error.getMessage(), exception);
        }
    }
```

如果您抛出的异常本身是一个 GraphqlError，那么它将把消息和自定义扩展属性从该异常转移到 exceptionwhiledatatfetching 对象中。这允许您将自己的定制属性放置到发送回调用者的 graphql 错误中。

例如，假设您的数据读取器抛出此异常。Foo 和 fizz 属性将包含在结果 graphql 错误中。

```java
    class CustomRuntimeException extends RuntimeException implements GraphQLError {
        @Override
        public Map<String, Object> getExtensions() {
            Map<String, Object> customAttributes = new LinkedHashMap<>();
            customAttributes.put("foo", "bar");
            customAttributes.put("fizz", "whizz");
            return customAttributes;
        }

        @Override
        public List<SourceLocation> getLocations() {
            return null;
        }

        @Override
        public ErrorType getErrorType() {
            return ErrorType.DataFetchingException;
        }
    }
```

> 翻译到这里先停一下吧，单纯的翻译只是浪费时间。前段时间项目紧，没时间吃透这个graphql，所以现在想直接从个人网盘项目里面尝试一下。