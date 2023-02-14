---
title: GraphQl-Java(一)Schema
date: 2021-01-19 10:13:53
tags: [GraphQL]
description: 本文基于GraphQl-java:v1.6版本官方文档中文翻译。
---

## 引入依赖

[版本列表](https://github.com/graphql-java/graphql-java/releases)

### Gradle

```grovvoy
repositories {
    mavenCentral()
}

dependencies {
    compile 'com.graphql-java:graphql-java:15.0'
}
```

### Maven

```xml
<dependency>
    <groupId>com.graphql-java</groupId>
    <artifactId>graphql-java</artifactId>
    <version>15.0</version>
</dependency>
```

## Schema——大纲

创建一个Schema有两种方式：

1. 推荐SDL(special graphql dsl):
    ```sdl
    type Foo {
        bar: String
    }
    ```

2. java code:
    ```java
    GraphQLObjectType fooType = newObject()
        .name("Foo")
        .field(newFieldDefinition()
                .name("bar")
                .type(GraphQLString))
        .build();
    ```



## DataFetcher and TypeResolver

`DataFetcher`用来向field提供数据，每一个`field`都包含一个DataFetcher,默认采用`PropertyDataFetcher`。

`PropertyDataFetcher` 从`Map`或者`Java Bean`中获取数据，所以当field name与map的key，或者类的元素名称的相匹配，就不需要定义`DataFetcher`了。

`TypeResolver`帮助`graphql-java`判断值的类型。

想象一下，你有一个接口MagicUserType，用来解析一系列java类Wizard,Witch和Necromancer。类型检查器负责检查一个运行时对象，并且判断用哪个`GraphqlObjectType`去响应这个对象，对应的应该用哪个`data fetchers`和field去运行。

```java
new TypeResolver() {
    @Override
    public GraphQLObjectType getType(TypeResolutionEnvironment env) {
        Object javaObject = env.getObject();
        if (javaObject instanceof Wizard) {
            return env.getSchema().getObjectType("WizardType");
        } else if (javaObject instanceof Witch) {
            return env.getSchema().getObjectType("WitchType");
        } else {
            return env.getSchema().getObjectType("NecromancerType");
        }
    }
};
```

## 用SDL创建一个Schema

当通过schema创建一个SDL，你应该定义好`DataFetcher`和`TypeResolver`。

比如创建一个starWarsSchema.graphqls:

>这货是星战迷，日了，好多电影里自创的名词。幸亏前阵子看了《曼达洛人》，要不然都翻译不了这破官文。

```sdl
schema {
    query: QueryType
}

type QueryType {
    hero(episode: Episode): Character //英雄
    human(id : String) : Human  //人类
    droid(id: ID!): Droid   //德鲁伊
}

//BGM枚举
enum Episode {
    NEWHOPE //新希望军
    EMPIRE  //帝国军
    JEDI    //绝地武士
}

//角色接口
interface Character {
    id: ID!
    name: String!
    friends: [Character]
    appearsIn: [Episode]! //出现的时候播放对应的BGM
}

type Human implements Character {
    id: ID!
    name: String!
    friends: [Character]
    appearsIn: [Episode]!
    homePlanet: String  //母星
}

type Droid implements Character {
    id: ID!
    name: String!
    friends: [Character]
    appearsIn: [Episode]!
    primaryFunction: String //超能力
}
```

这个静态文件`starWarsSchema.graphqls`包含了field和type的定义，但是你还需要一个`runtimeWiring（运行时注入）`来把这个静态文件转换为一个可运行的schema。

`runtimeWiring`必须包含`DataFetcher`、`TypeResolver`和自定义的`Scalar`。

你可以用下面的建造者模式+lambda表达式去创建报文：

```java
 RuntimeWiring buildRuntimeWiring() {
    return RuntimeWiring.newRuntimeWiring()
            .scalar(CustomScalar)
            // this uses builder function lambda syntax
            .type("QueryType", typeWiring -> typeWiring
                    .dataFetcher("hero", new StaticDataFetcher(StarWarsData.getArtoo()))
                    .dataFetcher("human", StarWarsData.getHumanDataFetcher())
                    .dataFetcher("droid", StarWarsData.getDroidDataFetcher())
            )
            .type("Human", typeWiring -> typeWiring
                    .dataFetcher("friends", StarWarsData.getFriendsDataFetcher())
            )
            // you can use builder syntax if you don't like the lambda syntax
            .type("Droid", typeWiring -> typeWiring
                    .dataFetcher("friends", StarWarsData.getFriendsDataFetcher())
            )
            // or full builder syntax if that takes your fancy
            .type(
                    newTypeWiring("Character")
                            .typeResolver(StarWarsData.getCharacterTypeResolver())
                            .build()
            )
            .build();
}
```

最终，你可以把静态的大纲和注入结合到一起来创建一个可运行的schema：

```java
SchemaParser schemaParser = new SchemaParser();
SchemaGenerator schemaGenerator = new SchemaGenerator();

File schemaFile = loadSchema("starWarsSchema.graphqls");

TypeDefinitionRegistry typeRegistry = schemaParser.parse(schemaFile);
RuntimeWiring wiring = buildRuntimeWiring();
GraphQLSchema graphQLSchema = schemaGenerator.makeExecutableSchema(typeRegistry, wiring);
```

另外在建造者模式的基础上，`TypeResolver`和`DataFetcher`可以使用`WiringFactory`接口来注入。这会允许更多的动态运行时，通过schema的定义去判断应该注入什么东西。你可以通过阅读SDL来判断生成哪种runtime。

```java
RuntimeWiring buildDynamicRuntimeWiring() {
    WiringFactory dynamicWiringFactory = new WiringFactory() {
        @Override
        public boolean providesTypeResolver(TypeDefinitionRegistry registry, InterfaceTypeDefinition definition) {
            return getDirective(definition,"specialMarker") != null;
        }

        @Override
        public boolean providesTypeResolver(TypeDefinitionRegistry registry, UnionTypeDefinition definition) {
            return getDirective(definition,"specialMarker") != null;
        }

        @Override
        public TypeResolver getTypeResolver(TypeDefinitionRegistry registry, InterfaceTypeDefinition definition) {
            Directive directive  = getDirective(definition,"specialMarker");
            return createTypeResolver(definition,directive);
        }

        @Override
        public TypeResolver getTypeResolver(TypeDefinitionRegistry registry, UnionTypeDefinition definition) {
            Directive directive  = getDirective(definition,"specialMarker");
            return createTypeResolver(definition,directive);
        }

        @Override
        public boolean providesDataFetcher(TypeDefinitionRegistry registry, FieldDefinition definition) {
            return getDirective(definition,"dataFetcher") != null;
        }

        @Override
        public DataFetcher getDataFetcher(TypeDefinitionRegistry registry, FieldDefinition definition) {
            Directive directive = getDirective(definition, "dataFetcher");
            return createDataFetcher(definition,directive);
        }
    };
    return RuntimeWiring.newRuntimeWiring()
            .wiringFactory(dynamicWiringFactory).build();
}
```

## 创建一个schema定义

当schema生成时，`DataFetcher`和`TypeResolver`可以在类型创建时提供：

```java
DataFetcher<Foo> fooDataFetcher = new DataFetcher<Foo>(){

    @Override
    public Foo get(DataFetchingEnvironment environment) {
        // environment.getSource() is the value of the surrounding
        // object. In this case described by objectType
        Foo value = perhapsFromDatabase(); // Perhaps getting from a DB or whatever
        return value;
    }
};

GraphQLObjectType objectType = newObject()
        .name("ObjectType")
        .field(newFieldDefinition()
                .name("foo")
                .type(GraphQLString)
        )
        .build();

GraphQLCodeRegistry codeRegistry = newCodeRegistry()
        .dataFetcher(
                coordinates("ObjectType", "foo"),
                fooDataFetcher)
        .build();
```

## GraphQl支持的类型

- Scalar——标量
- Object——对象
- Interface——接口
- Union——联合
- InputObject——输入对象
- Enum——枚举

### Scalar

graphql-java支持下面几种标量：

#### 标准标量：

- GraphQLString
- GraphQLBoolean
- GraphQLInt
- GraphQLFloat
- GraphQLID

#### 扩展标量：

- GraphQLLong
- GraphQLShort
- GraphQLByte
- GraphQLFloat(怎么重复了？)
- GraphQLBigDecimal
- GraphQLBigInteger

**注意：你的客户端可能不能理解扩展标量的语意，比如把Java中的long(最大值26^3-1)匹配给JavaScript的Number类型(最大值2^53 - 1)可能会出问题。

>这里用了两个may be，我觉得身为一个官文，这样写是不对的。会就是会，may be什么鬼。

### Object

SDL Example:

```sdl
//Simpson应该是卡通片《辛普森一家》中的一类怪物
type SimpsonCharacter {
    name: String
    mainCharacter: Boolean //是否是主角
}
```

Java Example:

```java
GraphQLObjectType simpsonCharacter = newObject()
    .name("SimpsonCharacter")
    .description("A Simpson character")
    .field(newFieldDefinition()
            .name("name")
            .description("The name of the character.")
            .type(GraphQLString))
    .field(newFieldDefinition()
            .name("mainCharacter")
            .description("One of the main Simpson characters?")
            .type(GraphQLBoolean))
    .build();
```

### Interface

Interfaces 是类型的抽象定义，哇塞！

SDL Example:

```sdl
//滑稽角色
interface ComicCharacter {
    name: String;
}
```

Java Example:

```java
GraphQLInterfaceType comicCharacter = newInterface()
    .name("ComicCharacter")
    .description("An abstract comic character.")
    .field(newFieldDefinition()
            .name("name")
            .description("The name of the character.")
            .type(GraphQLString))
    .build();
```

### Union

SDL Example:

```sdl
type Cat {
    name: String;
    lives: Int;
}

type Dog {
    name: String;
    bonesOwned: int;
}

union Pet = Cat | Dog
```

>我擦，《猫狗大战》

Java Example:

```java
TypeResolver typeResolver = new TypeResolver() {
    @Override
    public GraphQLObjectType getType(TypeResolutionEnvironment env) {
        if (env.getObject() instanceof Cat) {
            return CatType;
        }
        if (env.getObject() instanceof Dog) {
            return DogType;
        }
        return null;
    }
};
GraphQLUnionType PetType = newUnionType()
        .name("Pet")
        .possibleType(CatType)
        .possibleType(DogType)
        .build();

GraphQLCodeRegistry codeRegistry = newCodeRegistry()
        .typeResolver("Pet", typeResolver)
        .build();
```

### Enum

SDL Example:

```sdl
enum Color {
    RED
    GREEN
    BLUE
}
```

Java Example:

```java
GraphQLEnumType colorEnum = newEnum()
    .name("Color")
    .description("Supported colors.")
    .value("RED")
    .value("GREEN")
    .value("BLUE")
    .build();
```

### ObjectInputType

SDL Example:

```sdl
input Character {
    name: String
}
```

Java Example:

```java
GraphQLInputObjectType inputObjectType = newInputObject()
    .name("inputObjectType")
    .field(newInputObjectField()
            .name("field")
            .type(GraphQLString))
    .build();
```

## 类型关系和递归

GraphQL支持递归（哇塞！）：比如一个`Person`类内部含有一个`List<Friend>`，我们可以声明一个类`graphql-java`并持有一个`GraphQLTypeReference`类型的属性。当schema创建时，`GraphQLTypeReference`会被实际的类型替换掉。(答不对题，文彩太差了)。

例如：

```java
GraphQLObjectType person = newObject()
    .name("Person")
    .field(newFieldDefinition()
            .name("friends")
            .type(GraphQLList
            .list(GraphQLTypeReference.typeRef("Person"))
            )
        )
        .build();
```

当schema通过SDL创建时，不需要对递归类型做特殊处理，递归已经自动完成了。

>所以你就可以不给出个SDL的例子吗，作者？

## Schema SDL 的模块化

一个巨大的schema文件是不方便浏览的（我想起了操哥写过的300行的函数和嵌套了7层的if）。我们可以通过两种技术使之**模块化**。

NO.1 在逻辑单元(java代码)中合并多个Schema SDL文件。

```java
SchemaParser schemaParser = new SchemaParser();
SchemaGenerator schemaGenerator = new SchemaGenerator();

File schemaFile1 = loadSchema("starWarsSchemaPart1.graphqls");
File schemaFile2 = loadSchema("starWarsSchemaPart2.graphqls");
File schemaFile3 = loadSchema("starWarsSchemaPart3.graphqls");

TypeDefinitionRegistry typeRegistry = new TypeDefinitionRegistry();

// each registry is merged into the main registry
typeRegistry.merge(schemaParser.parse(schemaFile1));
typeRegistry.merge(schemaParser.parse(schemaFile2));
typeRegistry.merge(schemaParser.parse(schemaFile3));

GraphQLSchema graphQLSchema = schemaGenerator.makeExecutableSchema(typeRegistry, buildRuntimeWiring());
```

NO.2 Graphql SDL的类型系统针对多模块具备了另一个构造函数，你可以使用`type extensions`去把额外的field和insterfaces添加到类型中。

想象你开始的时候写了这么个type：

```sdl
type Human {
    id: ID!
    name: String!
}
```

在另一部分文件中可以通过extend这个type来添加更多的图形。

```sdl
extend type Human implements Character {
    id: ID!
    name: String!
    friends: [Character]
    appearsIn: [Episode]!
}
```

你可以添加更多文件，它们最终会被整合到一起（不允许重复定义字段）。

```sdl
extend type Human {
    homePlanet: String
}
```

最终当所有type的继承者汇集到一起时，在运行时中type会变成这个样子：

```sdl
type Human implements Character {
    id: ID!
    name: String!
    friends: [Character]
    appearsIn: [Episode]!
    homePlanet: String
}
```

这在顶级中很有用，你可以使用继承来向顶级的schema「query」中添加新的字段。团队开发中可以采用这种形式来向所有的query默认提供顶级query。

```sdl
schema {
    query: CombinedQueryFromMultipleTeams
}

type CombinedQueryFromMultipleTeams {
    createdTimestamp: String
}

# maybe the invoicing system team puts in this set of attributes
extend type CombinedQueryFromMultipleTeams {
    invoicing: Invoicing
}

# and the billing system team puts in this set of attributes
extend type CombinedQueryFromMultipleTeams {
    billing: Billing
}

# and so and so forth
extend type CombinedQueryFromMultipleTeams {
    auditing: Auditing
}
```

## 订阅支持(哇塞！)

订阅允许你跑个query，并且持续监听这个query返回对象的变化。（这个牛逼了，但是有可能是靠长链接维持的，如果是的话开销就有点大了）

```sdl
subscription foo {
    # normal graphql query
}
```

具体请查看[Subscriptions](https://www.graphql-java.com/documentation/v16/subscriptions)。

## 后记

终于翻译完了，作者文彩比较渣，所以翻译的有点累。下一篇准备翻译这个`Subscription`，看看它的订阅到底是怎么回事。当然在此之前，需要对本篇翻译二次回顾，消化吸收。所以下一篇翻译不知道什么时候能搞出来了。

---

翻译完之后的第一次校稿，发现单纯阅读这个Schema篇根本理解不了。原因是按照demo跑完入门代码后，并没有好好阅读理解逻辑，然后中间因为来活了，中断了学习和翻译，特么尴尬了。下一篇还是从quick start开始吧。