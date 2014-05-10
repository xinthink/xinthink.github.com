---
layout: post
title: "Promises风格的异步编程"
description: "Promises风格的异步编程"
category: node
tags: node.js async promise callback fp
path: _posts/node/2014-04-10-promised-async-prog.md
---
## Callback Hell
异步I/O是Node.js的卖点，在处理高并发I/O的场景时确实卓有成效，但世上没有免费的晚餐（因为我知道某歌有免费午餐），作为交换，你必须付出改变既有编程习惯的代价。

比如，在其他编程环境中，这样的代码是很常见的

```java
try {
  Object v1 = doIO1();  // 第一个I/O操作
  return doIO2(v1);     // 第二个I/O操作，需要依赖之前的输出
}
catch (Exception e) {
  ...                   // 异常处理
}
```

但是对不起，在Node.js中，你只能放弃这种写法了([Generators](#generators)除外)，在不使用辅助lib的情况下，你需要这么写：

```javascript
doIO1(function(err, v1) {        // 第一个I/O操作
  if (err) {
    ...                          // 错误处理
    return
  }

  doIO2(v1, function(err, v2) {  // 第二个I/O操作，需要依赖之前的输出
    if (err) {
      ...                        // 错误处理
      return
    }

    ...                          // 最终结果，可通过函数调用或事件传递出去
  });
});
```

最终，层层嵌套的回调函数会使得代码难以阅读和维护，这就是所谓的[Callback Hell](http://callbackhell.com)。

[@jcoglan](https://github.com/jcoglan)甚至认为采用了回调函数风格是Node.js的一项重大决策失误，详见 <http://bit.ly/1hDGL4g> (另，此文对指令式和函数式编程的观点令人印象深刻)

对于如何拆解Callback Hell，已有很多强文，请Google之。以下仅从个人体会出发，针对一些貌似大家提的不多的方面补充自己的看法。


## Async.js遗忘的角落
为了解决这个问题，我开始使用[Async.js](https://github.com/caolan/async)

```javascript
var doIO1AndIO2 = async.compose(doIO2, doIO1);  // 组合两个异步函数

doIO1AndIO2(function(err, v2) {
  if (err) {
    ...                        // 错误处理
    return
  }

  ...  // 得到最终结果，处理之
});
```

很酷，对不对？Async.js真是一个伟大的lib，但是即便如此，仍然有它触及不到的角落...

假设我有一个同步函数`do1`，它返回一个值，如果我想把`do1`和`doIO2`组合起来，怎么办？`async.compose`和`underscore.compose`都办不到，它们一个只适用于回调风格，另一个则只适用于返回值风格...

有人会说：切，`doIO2(do1(), function(err, v2) {...`不就完了？是的，一般情况下这都是不错的选择。但如果我们希望最大程度地重用逻辑时，让函数保持独立，根据实际需要灵活组合就是更好的选择了。所以，将同步和异步函数组合起来使用的场景是可能存在的。

这时我们就需要一个异步版本的`do1`：

```javascript
// 伪异步版do1
function do1AndCallback(cb) {
  ...
  cb(null, v1);
}

// 然后组合之
var do1AndIO2 = async.compose(doIO2, do1AndCallback);
```

但是（又来了），既然我们说过希望函数能够灵活组合，那么`do1`就可能需要和其他同步函数进行组合，所以我们既需要一个同步版的`do1`，也需要一个伪异步版的`do1`，即`do1AndCallback`，嗯，坏味道...

请记住这个瑕疵，我们回头再来讨论。


## 第二条路：Promises

另一个解决Callback Hell的途径是[Promises](http://www.promisejs.org/)，这是一套异步编程[模式](http://www.promisejs.org/patterns/)，引用[@jcoglan](https://github.com/jcoglan)的观点：Promise更具有函数式编程的特点，因为它让我们重新聚焦在value上。

* Promise有不少优秀的实现，以下的例子均基于其中的一个：[Q](https://github.com/kriskowal/q)

让我们以Promise风格，重写上文中的例子。

首先假设我们已经将`doIO1`、`doIO2`重写为Promised版本（就是为了让下面的代码更好看，具体实现请参考[Q文档](https://github.com/kriskowal/q)）

在此前提下，上述例子可以重写如下：

```javascript
var useResult   = function (v2) { ... };   // 得到最终结果，处理之
var handleError = function (err) { ... };  // 统一处理throw或callback传递出来的异常

doIO1().then(doIO2).then(useResult, handleError);
```

哇！很像命令行的pipe吧：`doIO1 | doIO2 | useResult`。我们只是描述了数据（value）的流向，而不必关注数据到底是如何成功地由1转到2的。这就是函数式编程的特点，现在稍微可以理解 [@jcoglan](https://github.com/jcoglan) 的说法了吧？

我并不打算详细介绍Promise，就此打住，具体请参考<http://www.promisejs.org>，以及其实现，[Q](https://github.com/kriskowal/q)。

总体来说，采用Promise风格的异步编程，可以使得：

* 异步过程成为value，可以被传递、被返回（还是那句话：更函数式）
* 函数的组合更为方便，可以灵活定义串行、并行处理的流程
* Exception更容易统一处理


## 同步异步的混搭

简单了解Promise以后，让我们来看看它是否能解决前文提到的同步异步混搭问题。

事情比想象中还要简单：

```javascript
Q.fcall(do1).then(doIO2).then(useResult, handleError);
```

现在，我们不再需要两个版本的`do1`，按照需要组合使用即可

```javascript
// 组合同步函数
do1And2 = _.compose(do2, do1);

// 异步 + 同步串行
doSthAsync().then(do1).then(do2).then...

// 并行
Q.all([
  Q.fcall(doSthSync),
  doSthAsync()
]).then...
```

当然，例子中的函数我写得很随意，实际上，函数能否组合在一起，需要考虑它们的参数列表及返回值是否匹配。


## 最后

通过以上对比，我们可以了解到，Promise风格的异步编程要比回调风格更容易写出简练的代码，可以将我们从复杂的异步控制流中解放出来，重新专注在程序的核心价值，数据（Value），以及数据的流动、转换上。

这里选取的同步+异步函数组合场景，我在实践中确有应用，同时也觉得比较适合用来展示Promise编程的魅力，在此分享给大家。

<a id="generators"></a>最后的最后，前面提到的Generators，是一项在Node.js 0.11.x后提供的特性。

简单讲，就是让我们可以像写阻塞性代码（如文章开头那段Java代码）那样调用异步过程。请参考： <https://github.com/visionmedia/co#readme>
