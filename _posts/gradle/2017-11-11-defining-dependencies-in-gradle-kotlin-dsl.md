---
layout: post
title: "Defining Dependencies in Gradle Kotlin DSL"
subtitle: "A concise syntax to define reusable dependencies"
feature: /images/feature-gradle-kts.svg
category: [gradle, kotlin-dsl, extensions]
tags:
path: _posts/gradle/2017-11-11-defining-dependencies-in-gradle-kotlin-dsl.md
excerpt: |
  In a multi-module Gradle project like most Android projects, you may want to manage all the dependencies in one place, keeping them consistent across the whole project.
  In this article, we'll archive this goal by defining a simple DSL (domain-specific language) using Kotlin extensions.
---

![Gradle + Kotlin = AwesomeThe](/images/feature-gradle-kts.svg)

## Motivation

If you have a multi-module [Gradle] project (e.g. most Android projects), you may want to define all the dependencies in one place, then refer to them in sub-projects' `build.gradle` file, making it easier to maintain the dependencies across the whole project.

This approach is exemplified by Jake Wharton's [U+2020 app], in which dependencies are pre-defined and stored into [Extra Properties] as a map:

```gradle
// global definition in root project
ext.versions = [
  compileSdk: 26,
  targetSdk: 25,
  …
]
ext.deps = [
  support: [
    appCompat: “com.android.support:appcompat-v7:${versions.supportLibrary}”,
    design: “com.android.support:design:${versions.supportLibrary}”,
    …
  ],
  picasso: ‘com.squareup.picasso:picasso:2.5.2’,
  …
]
```

then they will be retrieved from inside sub-projects:

```gradle
dependencies {
  implementation deps.support.appCompat
  implementation deps.support.design
  …
}
```

It's straightforward for Groovy-flavored Gradle scripts. However, it'll be a bit different when it comes to [Gradle Kotlin DSL] (i.e. Gradle scripts written in [Kotlin]).

## The Problem

Kotlin compiler, which is statically typed, will complain about the following statement, which is the Kotlin equivalent of the dependency reference we wrote in the snippets above.

![Kotlin Syntax Error](/images/gradle-kts-syntax-error.jpg)

In fact, it has to be something like this:

```kotlin
// Declarations
ext["deps"] = mapOf(
  "support" to mapOf(
    "appCompat" to "com.android.support:appcompat-v7:26.0.2",
    "design" to "com.android.support:design:26.0.2"
  ),
  "picasso" to "com.squareup.picasso:picasso:2.5.2"
)

// References
implementation(((ext["deps"] as Map<*, *>)["support"] as Map<*, *>)["design"]!!)
```

🙀🙈 Well, it's simply unacceptable!

## A Better Syntax

I have to do some hackings to save my days. Fortunately, [Kotlin] has already given us powerful weapons: [Extensions] & [Operator Overloading].

I'll show you the result first and then explain the solution. The following snippet is the resulted syntax to define dependencies:

```kotlin
// Dependency declarations
extra.deps {
  "support" {
    "appCompat"("com.android.support:appcompat-v7:26.0.2")
    "design"("com.android.support:design:26.0.2")
  }
  "picasso"("com.squareup.picasso:picasso:2.5.2")
}
```

This is the most concise syntax I can achieve. optionally, the dependencies can be grouped recursively, with no limit.

The following is how we retrieve the dependencies later, please notice that a dot (`.`) operator can be used to access grouped dependency, *square brackets* are also supported:

```kotlin
// Dependency retrieval
dependencies {
  compile(deps["support.appCompat"]) // use a `.` operator
  compile(deps["support"]["design"]) // use a `[]` operator
  compile(deps["picasso"])
}
```

Much better! 🎉

## Explanation
To achieve the above syntax, it's not that difficult actually, couples of operators & extensions are enough to get the job done.

Firstly, build a tree structure to store the dependency definitions:

```kotlin
interface DependencyItem

data class DependencyNotation(val notation: String) : DependencyItem

class DependencyGroup : DependencyItem {
  val dependencies: Map<String, DependencyItem>
}
```

Equips the structure with operators to make the magic happen:

```kotlin
class DependencyGroup {
  // provides the `.``[]` operators
  operator fun get(key: String): DependencyItem

  // provides the `<key>(<notation>)` syntax
  operator fun String.invoke(notation: String)

  // provides the `<key> {<group>}` syntax
  operator fun String.invoke(init: DependencyGroup.() -> Unit)
}
```

Finally, put the dependencies tree into [Extra Properties]:

```kotlin
val ExtraPropertiesExtension.deps: DependencyGroup
  get() =
    if (has("deps")) this["deps"] as DependencyGroup
    else DependencyGroup().apply {
      this@deps["deps"] = this
    }
```

That's all, it's done! 🍻 Now you have easy access to the dependencies by writing expressions like `deps["support.appCompat"]`.

## Conclusion

Actually, what we just built is a simple [DSL (domain-specific language)], which we use to manage the dependencies across multiple Gradle modules.

The completed code can be found in this [Gist][complete-gist]. Put them under the `buildSrc` directory to enjoy the syntax sugar.

> For those who are not familiar with `buildSrc`, please refer to the [Guide][build-src-guide].

I hope you enjoy the hacking, please let me know if you have any comments or better solutions. 🤝🖖


---

This article is originally published on [AndroidPub].


[Gradle]: https://gradle.org
[Gradle Kotlin DSL]: https://github.com/gradle/kotlin-dsl
[DSL (domain-specific language)]: https://en.wikipedia.org/wiki/Domain-specific_language
[U+2020 app]: https://github.com/JakeWharton/u2020/blob/master/build.gradle
[Extra Properties]: https://docs.gradle.org/current/dsl/org.gradle.api.plugins.ExtraPropertiesExtension.html
[build-src-guide]: https://docs.gradle.org/current/userguide/organizing_gradle_projects.html#sec:build_sources
[Kotlin]: https://kotlinlang.org
[Extensions]: https://kotlinlang.org/docs/reference/extensions.html
[Operator Overloading]: https://kotlinlang.org/docs/reference/operator-overloading.html
[complete-gist]: https://gist.github.com/xinthink/2e838366f65728ac0dbd68bdd2bb7168
[AndroidPub]: https://android.jlelse.eu/defining-dependencies-in-gradle-kotlin-dsl-8da748276e9e
