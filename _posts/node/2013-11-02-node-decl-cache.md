---
layout: post
title: "在 Node.js 中使用声明式缓存"
description: "在 Node.js 中使用声明式缓存"
category: node
tags: node.js fp
path: _posts/node/2013-11-02-node-decl-cache.md
---
## Why
写了多年的 Java 程序，即使在转投 [Node](http://nodejs.org) 之后，仍然对 [Spring 框架](http://projects.spring.io/spring-framework) 的 IoC 、Declarative XXX 记忆犹新，于是在 Node 项目中要用到缓存时，自然地想起了 [Declarative caching](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html#cache-annotations)，就有了山寨一把的想法。。。

### 问题
为什么要用缓存就不说了，地球猿都知道，直接看问题，假设我们有一个查询客户信息的函数：

```coffee-script
### Find customer from db by id.
  @param id [Integer|String] customer id
  @param cb [Function] callback (err, customer)
###
findCustomer = (id, cb) ->
  db.query "SELECT * FROM tbl_customers WHERE id = ?", [id], cb
```

* 本文的示例代码假设读者了解 [Node](http://nodejs.org) 及异步编程，甚至 [CoffeeScript](http://coffeescript.org), [Underscore.js](http://underscorejs.org), [Async.js](https://github.com/caolan/async#readme)
* 简单起见，示例代码忽略了数据库/Cache Server 连接的建立、释放，以及部分的异常处理

要对 customers 进行缓存，无非是先检查 cache ，命中则直接返回，否则查询数据库，先缓存结果然后返回。
最原始的实现大概是这样的：

```coffee-script
# The cached version of findCustomer.
findCustomerWithCache = (id, cb) ->
  fromCache "customer:#{id}", (err, cust) ->
    return cb err if err

    if cust  # cache hit
      cb null, cust
    else  # cache missed
      loadAndCacheCustomer id, cb
```

辅助函数定义:

```coffee-script
### Try to retrieve item with key k from cache.
  @param k [String] key of the cached item
  @param cb [Function] callback (err, v) v is the cached item, false if NOT found
###
fromCache = (k, cb) -> cache.get k, cb


### Save value v to cache with key k.
  @param dur [Integer] seconds before cache expires
  @param k [String] key of the cached item
  @param v [Any] item to be cached
  @param cb [Function] callback (err, v)
###
toCache = (dur, k, v, cb) -> cache.set k, v, dur, (err) -> cb err, v


# find customer from db, and cache it before return the result
loadAndCacheCustomer = (id, cb) ->
  _cacheCust = _.partial toCache, 86400, "customer:#{id}"  # caching for 1 day
  async.compose(_cacheCust, findCustomer) id, cb
```

很好，现在调用 `findCustomerWithCache` 函数就能够利用到缓存了。

但这个实现的问题是：

1. 凡是有一个需要缓存的 `findXXX` 函数，都要编写配套的 `findXXXWithCache` `loadAndCacheXXX`
2. 使用或不使用缓存，需要调用不同的函数，这意味着一旦切换，就要修改所有的函数调用处


## 利用高阶函数重用代码
首先考虑通过函数重用来解决第一个问题：如果有一个通用的函数能够代为处理缓存相关的事务，只在缓存没有命中时调用实际的查询过程，就达到目的了。

### API 设计
设计一个函数 (api) 最好的方法就是首先把期望的使用形式写出来。所以这个关于缓存的高阶函数，暂且命名为 `withCache` ，或许这么用起来会不错：

```coffee-script
# The cached version of findCustomer.
findCustomerWithCache = (id, cb) ->
  withCache 86400, ((id) -> "customer:#{id}"), findCustomer, id, cb
```

也就是说，我们只要把数据查询过程 `findCustomer` 委托给一个高阶函数 `withCache` ，并且设定缓存时长以及 key 的生成规则，就可以了，缓存检查、更新的整个流程都由这个函数封装，从而达到重用的目的。

### 封装缓存流程

```coffee-script
### The generic caching routine.
  @param dur [Integer] seconds before cache expires
  @param keyBuilder [Function] (args...) given the arguments, return a unique cache key
  @param fn [Function] the actual data loading routine
  @param args... arguments applys to fn to load data
  @param cb [Function] callback (err, data) applys to fn to receive data
###
withCache = (dur, keyBuilder, fn, args..., cb) ->
  key = keyBuilder args...  # compute cache key

  fromCache key, (err, data) ->
    return cb err if err

    if data  # cache hit
      cb null, data
    else  # cache missed
      loadAndCacheData dur, key, fn, args..., cb


### Load from db using fn.
  and cache it using the given key before return the result.
###
loadAndCacheData = (dur, key, fn, args..., cb) ->
  _cacheData = _.partial toCache, dur, key
  async.compose(_cacheData, fn) args..., cb

```

一旦缓存流程被封装起来，重用就变得容易了，现在我们可以对任意的数据查询函数应用缓存了。

这也是函数式编程重用代码的一种最基本的方法，用高阶函数封装过程，就像 `each` `map` `reduce` 等函数对集合应用的封装一样。

注意到 `withCache` 函数的签名，参数的排列顺序不是随意的，而是考虑了它们的适用范围 (或稳定程度)，使得函数重用更为方便，现在很容易做到：

```coffee-script
withOneDayCache = _.partial withCache, 86400

# the same as: withCache 86400, ((id) -> "customer:#{id}"), findCustomer, id, cb
withOneDayCache ((id) -> "customer:#{id}"), findCustomer, id, cb

# exactly the same as the findCustomerWithCache function above
findCustomerWithOneDayCache = _.partial withOneDayCache, ((id) -> "customer:#{id}"), findCustomer
```


## 函数代理
现在我们距离声明式缓存仅一步之遥，不过在此之前，我们还可以再进一步：我们想要以更接近‘声明式’地获取 `findCustomerWithOneDayCache` 这样的函数。

以 [Spring Declarative annotation-based caching](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html#cache-annotations) 为参照：

```java
@Cacheable("books")
public Book findBook(ISBN isbn);
```

在 [Node](http://nodejs.org) 中，我们希望能够以以下方式‘声明’某个函数‘应该使用缓存’：

```coffee-script
findCustomerWithCache = cacheable findCustomer, ns: 'customer', dur: 86400
```

为便于理解，不妨将 `cacheable` 类比为 [Spring ProxyFactoryBean](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop-api.html#aop-pfb)，通过它，我们可以获得一个代理，其表现形式无异于原函数 (如本例中 `findCustomer`)，却能够在背后默默地帮我们完成缓存的相关处理。

为实现这一机制，我们首先需要一个通用的 cache key 生成方式:

```coffee-script
### Generates cache key using function arguments.
  @param ns [String] namespace, prefix of the cache key
  @param args [Array] arguments for loading data
###
getKey = (ns, args=[]) -> ns + ':' + ...
```

使用前缀 + 参数列表自动生成 cache key ，对于含有多个参数的情况，可以使用 hash (参考[Default Key Generation](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html#cache-annotations-cacheable-default-key))，这里我就不写了。

接下来就简单了，利用之前的成果，我们很容易就可以写出满足上述形式的 `cacheable` 函数：

```coffee-script
### Wrap the given function fn to use the caching service.
  @param fn [Function] (args..., cb) the actual data loading routine
  @param opts [Object] options
    ns:  [String] namespace, prefix of the cache key
    dur: [Integer] seconds before cache expires
###
cacheable = (fn, opts) -> _.partial withCache, opts, getKey, fn
```

当然， `withCache` 需要稍稍改动，接受一个 options 对象作为参数：

```coffee-script
withCache = (opts, keyBuilder, fn, args..., cb) ->
  ...
```


## 声明式缓存
好了，就差最后一步：如何能够在不影响代码的情况下，随意设置缓存参数 (是否缓存、缓存时长等)？

老办法，首先设定我们的期望，假设 `findCustomer` 是由 `customers` 模块提供的，可以这样使用:

```coffee-script
custs = require 'customers'
custs.findCustomer 1, (err, customer) -> ...
```

我们希望缓存对上述代码是透明的，即无论缓存与否或缓存多长时间，这段代码都不需要修改。

要解决这个问题，可以利用 [Node](http://nodejs.org) 模块的缓存特性，参考：[Node 文档](http://nodejs.org/api/modules.html#modules_caching)。
即，我们可以在程序一开始 (通常叫做 prelude)，将 `customers` 模块的 `findCustomer` 方法替换成 cacheable 的版本，只要 [Node](http://nodejs.org) 模块缓存没有被清空，随后所有对于该模块的 require 都将得到我们修订的版本， `findCustomer` 函数自然就是 cacheable 版了。

为此我们先准备一个函数，帮助我们替换模块中的方法：

```coffee-script
### Replace the function fn in a module m, to a cacheable one.
  @see http://nodejs.org/api/modules.html#modules_caching
  @param m [Object] module object
  @param fn [String] name of the function
  @param opts [Object] cache options
    ns:  [String] namespace, prefix of the cache key
    dur: [Integer] seconds before cache expires
###
cache = (m, fn, opts) -> m[fn] = cacheable m[fn], opts
```

现在我们建立一个 `caches.coffee` (当然可以是 .js，或其他任何名称)，重要的是它需要在 prelude 中得到处理。
可以把它类比为 Spring Bean 定义文件 (通常是 XML)，我们把缓存相关的‘声明’都放在这里：

```coffee-script
custs    = require 'customers'
whatever = require 'whatever'
...

cache custs,    'findCustomer',       ns: 'customer', dur: 86400
cache custs,    'findCustomerByName', ns: 'customer', dur: 86400
cache whatever, 'findWhatever',       ns: 'whatever', dur: 0
...
```

好了，现在我们可以‘声明’某个特定的函数是否以及如何使用缓存，随时修改这些‘声明’而不会对代码造成冲击。


## 最后
需要了解的是， [Spring Cache Abstraction](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html) 主要包括了缓存 api 的抽象，以便在不影响代码的情况下随意切换不同的缓存服务，本文讨论的更多是类似 [Declarative annotation-based caching](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html#cache-annotations) 的特性，因为相比于切换缓存服务，缓存配置变动的概率显然要大得多，更值得我们去仔细的管理，所以就不在此讨论 caching api 的抽象了。

另外，本文中讨论的实现方法依赖 [Node Module Caching](http://nodejs.org/api/modules.html#modules_caching) 特性 (即如果模块被解析为同一个文件，则永远得到同一个对象)，因此该方法有效的前提是：

1. 同一个模块不会被解析到不同的文件，详情参考[ Node 文档 ](http://nodejs.org/api/modules.html#modules_modules)
1. 模块缓存不会被清除

所以在应用这一方法之前，还请仔细评估上诉的前提条件。
