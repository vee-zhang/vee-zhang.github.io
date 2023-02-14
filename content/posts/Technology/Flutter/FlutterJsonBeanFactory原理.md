---
title: FlutterJsonBeanFactory原理
date: 2021-06-25 16:57:01
tags: [flutter,dart]
categories: [technology]
---

## 创建类

```dart
class StudentEntity with JsonConvert<StudentEntity> {

  @JSONField(name: "name")
  late String name;
}
```

## 使用

```dart
//序列化
Map<String, dynamic> json = StudentEntity(name: "走两步试试").toJson();

//反序列化方式一
final student1 = StudentEntity().fromJson(json);

//反序列化方式二
final student2 = JsonConvert.fromJsonAsT<StudentEntity>(json);

print('student1:$student1');
print('student2:$student2');
```

## 序列化过程

### JsonConvert

```dart
/// 当我们混入这个类时，就必须确定这个泛型
class JsonConvert<T> {

    ///序列化
    Map<String, dynamic> toJson() {

        //注意这里获取到了runtimeType
        return _getToJson<T>(runtimeType, this);
    }

    ///通过静态方法分发
    static _getToJson<T>(Type type, data) {
        switch (type) {
            case StudentEntity:
                //注意这里的强转
                return studentEntityToJson(data as StudentEntity);
            }
            return data as T;
        }
    }
}
```

这里首先拿到了`runtimeType`。每个dart对象都继承于Object：

```dart
@pragma("vm:entry-point")
class Object {
    
  @pragma("vm:recognized", "other")
  const Object();

  external bool operator ==(Object other);

  external int get hashCode;

  external String toString();

  @pragma("vm:entry-point")
  external dynamic noSuchMethod(Invocation invocation);

  external Type get runtimeType;
}
```

Object中有个get属性，而且加了`external`关键字修饰，表明dart会自动处理，不需要我们来实现。

接下来去看`studentEntityToJson(data)`方法，位于生成的`student_entity_helper.dart`中。

### 生成的help辅助文件

```dart
Map<String, dynamic> studentEntityToJson(StudentEntity entity) {
	final Map<String, dynamic> data = new Map<String, dynamic>();
	data['name'] = entity.name;
	return data;
}
```

完全就是遍历了StudentEntity文件中的属性，然后生成了这么个方法，一切的核心都是**强转**。这让我想起了号称最快的FastJson，原理也是强转。

## 反序列化

### JsonConvert

```dart
T fromJson(Map<String, dynamic> json) {
    return _getFromJson<T>(runtimeType, this, json);
}

static _getFromJson<T>(Type type, data, json) {
    switch (type) {
        case StudentEntity:
            return studentEntityFromJson(data as StudentEntity, json) as T; 
    }
    return data as T;
}
```

### 生成的help辅助文件

```dart
studentEntityFromJson(StudentEntity data, Map<String, dynamic> json) {
	if (json['name'] != null) {
		data.name = json['name'].toString();
	}
	return data;
}
```

不用猜，依然是强转，小学生的思维设计，甚至比不上fastJson考虑的多。

## 顶层列表处理

```dart
static M fromJsonAsT<M>(json) {
    //直接通过类型判断
    if (json is List) {
        return _getListChildType<M>(json);
    } else {
        return _fromJsonSingle<M>(json) as M;
    }
}

//单对象处理
static _fromJsonSingle<M>( json) {
    //看看，多简单的设计，但是思维还是挺巧妙的
    String type = M.toString();
    if(type == (StudentEntity).toString()){
        return StudentEntity().fromJson(json);
    }	
    return null;
}

//列表处理
static M _getListChildType<M>(List data) {
    if(<StudentEntity>[] is M){
        return data.map<StudentEntity>((e) => StudentEntity().fromJson(e)).toList() as M;
    }
    //啊，这个幼稚的异常
    throw Exception("not fond");
}
```

## 总结

看完以后，就觉得吧，我们的项目就是用这么个新手写就的插件上，奔跑了快一年时间。不是说这东西不好，他确实是用了简单直接的方式，解决了flutter编程上的一大痛点。而且当我刚刚接触flutter的时候，也觉得这个东西很方便，但是当时心里总是觉得不靠谱，所以没有采用。直到被别人强推了，那就用吧，然后就是磕磕绊绊的各种小问题，也都好解决，直到——当我需要部署devOps的时候，我惊喜的发现jenkins不能alt+j啊！

通过了解源码，也感觉原作者虽然是个编程新手，但是他在突破问题的时候确实是下了一番功夫的，而且他的知识面也比较宽，甚至通过idea插件的方式解决问题，也是很厉害的了。

最主要的是，确实解决了flutter对于json解析不方便的痛点，所以最后还是对作者表示：非常感谢！