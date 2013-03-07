# 拥抱Gradle: 下一代构建工具

## 认识Gradle
过去Java世界的人谈起构建，Ant、Maven一定是必备的词汇吧，而如今，‘Gradle’这个名字也渐渐吸引了更多人的目光。今天我们就来认识一下这位号称‘下一代构建工具’的Gradle。

不过，在此之前，我们先来温习一下既熟悉又陌生的Ant。

假设我们希望在编译Snapshot版本的时候打开debug开关，否则关闭，在Ant中可以用Condition实现：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project name="demo" default="compile">
  <property name="version" value="1.0-SNAPSHOT"/>

  <target name="compile">
    <condition property="debug" else="off">
        <contains string="${version}"
          substring="SNAPSHOT"/>
    </condition>
    <javac debug="${debug}"
      ...
    />
  </target>
</project>
```

嗯，我认为这段脚本不够直观，构建逻辑被淹没在一堆标签中，Condition的表达方式有点拗口。哦，你觉得挺好的，习惯了就好？好吧，看过Gradle的版本再下结论不迟：

```gradle
apply plugin: 'java'

version = '1.0-SNAPSHOT'

compileJava {
  options.debug = version.endsWith('SNAPSHOT')
}
```
这段代码更为简短，这点是毫无疑问的，而且，Java程序员从中可以找到属性访问、赋值、方法调用等再熟悉不过的元素。请注意我用了‘代码’一词，没错，Gradle脚本实际上就是一段Groovy代码。

那么对于“我们为什么需要下一代构建工具”这个问题，从这两段脚本的差异，可以分析如下：

* 程序员更容易读懂编程语言，而不是XML，后者擅长作为一种数据交换格式，是给机器读的（如Web Service）
* 编程语言的强大能力应为构建过程所用，如上面代码中的 `endsWith` 方法，而无需舍近求远地使用 `<contains>`
* 更重要的是，Java程序员无需翻阅文档就能使用 `if` `else` `endsWith` ，而 `<if>` `<else>` `<contains>` 则不见得
* 额外地，动态语言以及DSL能够使得程序员更专注在构建逻辑上，脚本也更为简洁，而在这方面XML真是望尘莫及

## Gradle是什么
在对Gradle有了一点感性认识以后，现在我们可以来讨论：Gradle究竟是什么？

讨论这个问题，我们还是得从Gradle的前辈们说起，Ant是一个非常通用的构建工具，对你做什么和怎么做几乎没有任何约束，通用的代价是牺牲了特定领域的便捷性，于是Ivy为我们提供了统一管理外部依赖的能力，Maven则在同一个工具内提供了构建和依赖管理的能力，同时引入了惯例优先的理念，从而使得Java工程的构建变得便捷，也更规范。

这些先辈都很优秀，但也有着共同的弱点：它们都使用XML作为描述格式；它们的插件开发不是很便捷（XML与Java之间是存在鸿沟的）。

作为后起之秀，在吸收前辈们的精华并设法克服它们的弱点之后，就有了Gradle:

* 同时提供了构建和依赖管理的能力，而且其内核就分别是Ant和Ivy（实在没有重写的必要）
* 使用Maven、Ivy的资源库（Repository）而没有重新发明一个。你只是换了一种使用方式，而不需要重建它们
* 但也不强制你使用Maven、Ivy的XML描述符，你甚至可以使用一个普通的文件夹作为资源库，而仍可拥有依赖传递的特性（Transitive Dependency）
* 也提供了惯例优先的模式，同时惯例默认值的覆盖变得更为容易
* 使用Groovy作为构建语言，并且提供了一套DSL，便于阅读、编写的同时赋予了构建脚本强大的能力，插件的编写、构建逻辑的复用也变得更为容易

## Gradle实战
现在，我们就通过具体的例子来进一步认识Gradle。虽然被设计为通用的构建工具，但无疑Java仍是Gradle的核心领域，我们就从这里开始。

### Java工程构建
#### 第一个例子
回忆一下第一节中的例子，它很酷，但究竟是什么意思？现在我们来剖析一下：

```gradle
apply plugin: 'java' // 引用Java插件

version = '1.0-SNAPSHOT' // 定义项目的版本号

// 覆盖compileJava任务（在Java插件中定义）默认的编译选项
compileJava {
  options.debug = version.endsWith('SNAPSHOT')
}
```
声明需要引用Java插件，并覆盖了compileJava任务的惯例设置，就是这样！假设工程的目录结构符合惯例（与Maven相同）、不依赖第三方组件，那么它就已经可以正确地工作。

从这个简单的例子，我们可以了解一些Gradle的理念：

1. 在专注各个领域的插件（如Java）中封装构建所需的大部分（如果不是全部）工作，并提供合理的默认值，即惯例（如工程目录结构）
2. 使用者的主要任务是根据实际情况覆盖各种默认设置

* 提一句，我们对Gradle的使用以及扩展，都应该符合这些理念，以便更好的享受Gradle为构建带来的进步

#### 外部依赖
当然，一个不依赖第三方组件的Java项目恐怕永远只能是个Demo吧，现在我们就为项目加上一些有用的组件。

在build.gradle中的任意位置：
```gradle
// 声明项目使用的资源库
repositories {
  mavenCentral() 
}

// 依赖声明
dependencies {
  compile 'org.slf4j:slf4j-api:1.6.6'

  runtime (
    'ch.qos.logback:logback-classic:1.0.7',
    'org.slf4j:jcl-over-slf4j:1.6.6',
  )

  testCompile 'junit:junit:4.10'
}
```
`compile`, `runtime` 和 `testCompile` 被称为Configuration，就是一组依赖的集合，它们之间存在着关联关系，比如 `compile` 中的依赖必然也会被包括在 `testCompile` 和 `runtime` 中。

在不同的环境下工作，就会使用不同的Configuration，从而检查出代码中可能存在的错误引用或者冗余引用，以及避免把测试组件发布到生产环境。

#### Properties

### 多工程构建
### 其他语言构建

### 依赖管理

* 引用
* 发布

### 插件

* built in/3'rd party
* 构建逻辑重用

### Gradle Wrapper

## 对比

* ant/maven/sbt

## 小结
