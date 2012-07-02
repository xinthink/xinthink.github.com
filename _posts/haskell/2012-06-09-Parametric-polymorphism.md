---
layout: post
categories: fp haskell
tags: programming fp haskell
---
Parametric polymorphism is when a function's type signature allows various arguments to take on arbitrary types, but the types must be related to each other in some way.

For example, in Java one can write a function that accepts two arguments of any possible type. However, Haskell goes further by allowing a function to accept two arguments of any type so long as they are both the same type. For example

...