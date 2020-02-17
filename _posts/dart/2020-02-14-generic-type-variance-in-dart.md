---
layout: post
title: "Generic Type Variance in Dart"
feature: /images/feature-dart-generic-type-variance.jpg
category: [dart, generics, variance]
tags:
path: _posts/dart/2020-02-14-generic-type-variance-in-dart.md
excerpt: |
  在泛型（generics）编程中，复杂的父/子类型关系判断常常带来困惑，比如何时List&lt;String>可以被看作List&lt;Object>的子类型？往一个List&lt;Object>类型的列表插入字符串对象是否总是被允许的？
  从C#到Kotlin，很多编程语言都支持类型「变体」(或型变，variance) 的特性，Dart在未来的版本中也会加入「声明处型变」的支持。在这篇学习笔记中，梳理了几种变体的含义，以及它们在Dart中的实现情况。
---

![Generic type variance in Dart](/images/feature-dart-generic-type-variance.jpg)

从C#到Kotlin，很多编程语言都支持类型「变体」(或型变，variance) 的特性，Dart在未来的版本中也会加入「声明处型变」(declaration-site variance) 的支持。
在这篇学习笔记中，梳理了几种变体的含义，以及它们在Dart中的实现情况。

---

[Dart]支持[泛型编程（generics）][Dart generics]特性，带来便利的同时也带来了相应的复杂度。比如应该如何判定两个泛型类型之间父/子类型关系？例如，以下语句通常会被认为是合理的：

```dart
List<int> integers = <int>[1, 2];
List<num> numbers = integers;
```

一组「整数」可以认为也是一组「数字」，看起来合情合理。然而，能够通过编译的语句：`numbers.add(2.0)`，却会导致运行时错误，因为`numbers`实际上具有`List<int>`类型，并不能接纳一个浮点数。

于是就需要引入类型参数**变体**（*variance*），进一步约束泛型的类型参数，使得编译器能够更好的帮助我们避免不必要的类型错误。问题在编译时发现，修复的成本将会比运行时才能发现的错误低很多，这也是静态类型语言的优势。

## 变体的种类

一共有三种类型的变体：
1. Covariance
  <br/>"*co-*" 词根一般有「共同、协同」的含义，意味着「同质、和谐」，因此这就是最直观的一种变体，例如：`Iterable<num> numbers = <int>[1, 2]`。即，如果类型参数具有子类型关系，就认为这两个泛型类型也具有子类型关系
1. Contravariance
  <br/>"*contra-*" 词根具有「相反」的含义, 因此它和*covariance*的赋值方向正好相反：`Writer<int> intWriter = Writer<Object>()`
1. Invariance
  <br/>"*in-*" 表示「否定」，这里表示不接受变体，只有类型严格相同，变量才可以相互赋值

## Producer / Consumer

上面这些不常见的英文单词容易混淆、晦涩难懂，笔者在看过几遍文档之后仍然难以清楚分辨（这也是写下这篇笔记的原因）。
通过参考[《Effective Java》]作者[Joshua Bloch]关于Java类型通配符的助记词：PECS，即，生产者-Extens，消费者-Super，我们会发现，借用「生产者、消费者」的概念同样有助于理解、记忆上述不同种类的变体。

### 生产者

纯粹的「生产者」只产出不消费，定义为*Covariance*，因为只产出，因此使用`out`关键字定义：

```dart
class Producer<out T> {
  T get next => …;
}

Producer<num> numbers = Producer<int>();
print(numbers.next);
```

`Producer<int>`生产`int`，而`int`是`num`的一种，因此把它理解为「`num`的生产者」是自然且安全的。

在*Covariance*约束之下，类型参数`T`将不能用于任何具有输入特征的地方：
```dart
class Producer<out T> {
  void add(T a) { … }
       ^ The 'out' type parameter 'T' can't be used in an 'in' position.
}
```

目前为止Dart的默认模式类似于*Covariance*，也就是说如果不使用`out`关键字，也可以这样写：

```dart
Iterable<num> numbers = <int>[1, 2];
numbers = numbers.map((x) => x * 2);
```
当然这样会缺少上述「out类型参数不能用于in的位置」的约束。

### 消费者

纯粹的「消费者」只负责消费而不产出，和「生产者」正好相反，因此属于*Contravariance*，用`in`关键字定义输入数据类型，例如：

```dart
class Consumer<in T> {
  void eat(T a) { … }
}
```

这时`Consumer`可以接收任何`T`类型的输入，这是最直观的传统用法，并不需要引入新语法，例如:

```dart
final consumer = Consumer<num>();
consumer.eat(1.0);
consumer.eat(2);
```

但是当我们需要限制输入数据的类型时，就是*contravariance*约束发挥作用的时候了：

```dart
final Consumer<int> consumer = Consumer<num>();
consumer.eat(10);
```

`Consumer<num>`接受`num`类型的输入，当然也能接受`int`类型的输入，所以使用一个`Consumer<int>`类型的变量来引用它是安全的。这样，编译器就能帮助我们避免输入不希望的数据类型，如`double`。

当然，在*Contravariance*约束之下，类型参数`T`将不能用于任何具有输出性质的地方：
```dart
class Consumer<in T> {
  final T first;
        ^ The 'in' type parameter 'T' can't be used in an 'out' position.
}
```

### 生产者 + 消费者

既输出又输入的情况就只能是*Invariance*了。为什么？

假设能够在一个类上同时使用*covariance*和*contravariance*，例如：

```dart
// Dart目前的默认模式，没有约束
class ReadWrite<T> {
  T read() {}
  void write(T a) {}
}
```

考虑以下两种场景：

```dart
// 1. covariance
ReadWrite<num> nrw = ReadWrite<int>();
nrw.write(0.0);
```
两个语句分开看都没问题，但`nrw`实际上只能接收`int`，而不应该允许输入`double`。

```dart
// 2. contravariance
ReadWrite<int> irw = ReadWrite<num>(); // 假设能够编译通过
int x = irw.read();
```
这里正好相反，我们认为`irw`输出`int`类型的数值，但实际上它却存在输出`double`的可能性，因此这种写法也应该禁止。

我们可以看到上面这两种场景是自相矛盾的，所以对于既输出又输入的情况，就只能限制类型参数必须相同了，定义关键字是`inout`：

```dart
class ReadWrite<inout T> {
  T read() {}
  void write(T a) {}
}
```

此时，上面例子中的两个赋值语句都会导致编译错误，实际上只能使用相同的类型参数去引用：

```dart
ReadWrite<int> irw = ReadWrite<int>();
irw.write(10);
int x = irw.read();

// 如果希望适用面更广，就需要使用父类了
ReadWrite<num> nrw = ReadWrite<num>()
  ..write(11)
  ..write(2.0);
num x = nrw.read();
```

## Use-site Variance

上面的例子可以看到*invariance*类型使用起来有太多限制，很不方便，比如：

```dart
void pipeline(ReadWrite<num> from, ReadWrite<num> to) {
  to.write(from.read());
}

// 不能这样做
pipeline(ReadWrite<int>(), ReadWrite<num>());
         ^ The argument type 'ReadWrite<int>' can't be assigned to the parameter type 'ReadWrite<num>'.
```
将`int`类型的数据作为`num`类型取出然后写入，这个合理的要求也被拒绝了。

为了更好的解决这个问题，就需要引入[使用处型变(use-site variance)][use-site variance]，可以理解为在使用时对数据类型做一些临时的约束（在允许范围内），以便满足处理的需求，这样就比较灵活了。

> 相对于这个概念，本文的上半部分实际上是对[声明处型变(declaration-site variance)][declaration-site variance]的说明，即在声明时对类型参数作出的限定


如果Dart支持*use-site variance*，应该可以类似这么写：
```dart
void pipeline(ReadWrite<out num> from, ReadWrite<num> to) {
  to.write(from.read());
}

// 允许
pipeline(ReadWrite<int>(), ReadWrite<num>());
```

意思就是限制了`pipeline`函数只能把`from`作为一个「生产者」来使用。这样编译器就会明白：此处将一个`ReadWrite<int>`赋值给一个`ReadWrite<num>`变量是安全的，编译就能够通过了，当然编译器也同时会禁止`from`去扮演一个消费者的角色。

不过目前*use-site variance*在Dart中还没有实现，上述问题需要这么写：
```dart
void pipeline<T extends num>(ReadWrite<T> from, ReadWrite<num> to) {
  to.write(from.read());
}
```

然而此时的编译器无法阻止`pipeline`函数作出`from.write(2.0)`这样的操作，将会导致运行时错误。
这类问题目前只能依靠开发者本身，通过Code review或者运行时错误分析去发现了，这也说明了*variance*特性的重要性。

## 现状

Dart的*declaration-site variance*特性目前处于实验阶段，为了体验这个特性，编译/运行时需要添加参数：`--enable-experiment=variance`。
[Analyzer][Dart analyer]则可以在`analysis_options.yaml`文件中配置：

```yaml
analyzer:
  enable-experiment:
    - variance
```

*use-site variance*特性的[GitHub issue][use-site variance]尚未被纳入任何Project，估计还需要等待比较长的一段时间才能够得到引入吧。

## 资料
- [Covariance and contravariance] on Wikipedia
- GitHub上的讨论: [declaration-site variance] & [use-site variance]
- [Type projections in Kotlin]


[Dart]: https://dart.dev/
[Dart generics]: https://dart.dev/guides/language/language-tour#generics
[Dart analyer]: https://dart.dev/guides/language/analysis-options
[Covariance and contravariance]: https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)
[declaration-site variance]: https://github.com/dart-lang/language/issues/524
[use-site variance]: https://github.com/dart-lang/language/issues/753
[Type projections in Kotlin]: https://kotlinlang.org/docs/reference/generics.html#type-projections
[《Effective Java》]: http://www.oracle.com/technetwork/java/effectivejava-136174.html
[Joshua Bloch]: https://en.wikipedia.org/wiki/Joshua_Bloch
