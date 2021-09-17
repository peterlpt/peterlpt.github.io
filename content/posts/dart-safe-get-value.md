---
title: "Dart语言安全取值"
categories: [Dart, Flutter]
tags: [Dart, extension-methods]
date: 2020-07-04T16:20:12+08:00
draft: false
---

充分利用Dart语言特性，对一般对象、List、Map简洁地进行级联安全取值的方法整理。

<!--more-->

### 1. 背景

对于Android Java出身的我，用java语言，在有限语法糖下，似乎没有特别的简洁写法：

```java
String x;
//case: 对象a持有对象b，对象b持有对象c，对象c持有对象d，对象d持有目标值
if(a!=null && a.b!=null && a.b.c!=null && a.b.c.d!=null){
  x = a.b.c.d.value;
} else {
  x = null;
}
```

需逐级判空，在项目开发的接口对接时，接口返回的数据相对复杂的节点动不动就有5、6层，要想安全取得目标值，码起来会令人抓狂。

这一切，在`Flutter`的`Dart`语言里，会变得简洁舒服很多。

### 2. `?.`操作符对于对象多层嵌套安全取值

`?.`操作符，当对象不为空时，执行`.`操作符访其成员。

针对上面的例子即可写成：

```dart
String x = a?.b?.c?.d?.value;
```

实际情况中，当a、b、c、d这四个对象每个都有几十个成员属性时，在业务接口数据解释时需要多次按需访问指定对象指定属性时，这种写法会越写越爽，不知不觉就爱上 `dart`爱上`Flutter`了，哈哈

直接在线[dartpad](https://dartpad.cn/)感受一下？

### 3. `?.`与`??`结合使用轻松指定缺省值

`??`操作符，当左边值为空时，取右边值。因此，如上述例子，有时我们有缺省值需求，比如：当拿不到值时，赋值123给x，利用`??`依然可一行简洁编写：

```dart
String x = a?.b?.c?.d?.value ?? '123';
```

### 4. `List`、`Map`如何简洁安全取值？

将上述例子细化调整如下：

```dart
class A{
  B b;
}
class B{
  List<C> cList = [C()];
}
class C{
  Map<String, dynamic> values = {'key': D()};
}
class D{
  String value = '456';
}
```

要从A的对象a中如何安全读取到d的value属性呢？可以这样写吗？：

```dart
void main(){
  var a = A();
  String x = a?.b?.cList[0]?.values['key']?.value;
}
```

这样写是不行的！运行会收到类似如下错误：

```shell
Unhandled exception:
NoSuchMethodError: The method '[]' was called on null.
Receiver: null
Tried calling: [](0)
#0      Object.noSuchMethod (dart:core-patch/object_patch.dart:51:5)
#1      main (package:flutter_layout/my_test.dart:16:25)
#2      _startIsolate.<anonymous closure> (dart:isolate-patch/isolate_patch.dart:301:19)
#3      _RawReceivePortImpl._handleMessage (dart:isolate-patch/isolate_patch.dart:168:12)
```

也就是说因为b为空，因此上面会变成直接尝试执行`null[0]`,显然null是无`[]`操作符的。

### 5. 利用`extension method`扩展List、Map实现级联简洁安全取值

在`dart`v2.7起，加入了[Extension methods](https://dart.dev/guides/language/extension-methods)支持，让上述需求可以继续简洁高效安全取值。

首先扩展`List`定义一个安全取值方法：

```dart
extension ListUtil<T> on List<T> {
  ///通过方法取值，而非[]操作符取值
  T safeGet(int index) {
    if (index >= 0 && index < length) {
      return this[index];
    }
    //如果越界则返回`null`
    return null;
  }
}
```

然后调整List的取值方式，不要使用`[]`操作符而使用新扩展的方法，此时就不会报错了：

```dart
void main(){
  var a = A();
  var c = a?.b?.cList?.safeGet(0);
  print(c); //此时即可如期打印出null，而不会抛错
}
```

类似的，扩展`Map`定义一个安全取值方法：

```dart
extension MapUtil<K, T> on Map<K, T> {
  ///通过方法取值，而非[]操作符取值
  T safeGet(K key) {
    return this[key];
  }
}
```

最后，针对调整后的例子及需求，可级联简洁安全取值如下：

```dart
void main(){
  var a = A();
  String x = a?.b?.cList?.safeGet(0)?.values?.safeGet('key')?.value;
  print(x); //如期打印输出null
  String x1 = a?.b?.cList?.safeGet(0)?.values?.safeGet('key')?.value ?? '123';
  print(x1); //打印输出默认值 123

  var aa = A();
  aa.b = B();
  aa.b.cList = [C()];
  String xx = aa?.b?.cList?.safeGet(0)?.values?.safeGet('key')?.value;
  print(xx); //打印输出目标值 456
}
```

### 6. 参考资料

1. [Extension methods](https://dart.dev/guides/language/extension-methods)
2. [Dart Operators](https://dart.dev/guides/language/language-tour#operators)、[Dart运算符](https://www.dartcn.com/guides/language/language-tour#运算符)
3. [使用Dart Extension，帮你扩充常用类的功能](https://blog.csdn.net/weixin_39649693/article/details/103618693)

