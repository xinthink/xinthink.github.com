---
layout: post
categories: fp haskell
tags: programming fp haskell
---
一个值如果根据上下文可以表现为多种类型，就可以称其之为多态的。多态可以分为很多不同的种类：

1. 参数型多态（[Parametric polymorphism](http://en.wikipedia.org/wiki/Parametric_polymorphism)），通常可以在函数式语言中找到
2. 特殊的多态（[Ad-hoc polymorphism  *  ](http://en.wikipedia.org/wiki/Ad-hoc_polymorphism)）或重载（Overloading）
3. 包含型多态（[Inclusion polymorphism](http://en.wikipedia.org/wiki/Inclusion_polymorphism)），多见于面向对象语言

  * 此处Ad-hoc指这种多态不属于类型系统的基本特性，因此取特例的含义

#### 例子

```haskell
foldr :: (a -> b -> b) -> b -> [a] -> b
```

foldr函数的类型包含了类型变量，因此它是一个参数型多态的函数。实际使用时，它可以用于任意类型的参数，比如：

```haskell
:: (Char -> Int -> Int) -> Int -> String -> Int -- a = Char, b = Int (note String = [Char])
:: (String -> String -> String) -> String -> [String] -> String -- a = b = String
```

数字文法是重载的（即为特殊的多态）：

```haskell
1 :: (Num t) => t
```
不同之处在于，此处的类型变量是受约束的 - 它必须是一个数字。


#### 参考资料

* [haskellwiki上的原文](http://www.haskell.org/haskellwiki/Polymorphism)
* [On Understanding Types, Data Abstraction, and Polymorphism (1985)](http://citeseer.nj.nec.com/cardelli85understanding.html), by Luca Cardelli, Peter Wegner in ACM Computing Surveys.
* [Type polymorphism](http://en.wikipedia.org/wiki/Type_polymorphism) at Wikipedia
