# 拥抱Gradle: 下一代构建工具

## 认识Gradle
过去Java世界的人谈起构建，Ant、Maven一定是必备的词汇吧，而如今，’Gradle‘这个名词也渐渐吸引了更多人的目光。今天我们就来认识一下这位号称’下一代构建工具‘的Gradle。

不过，在此之前，我们先来温习一下既熟悉又陌生的Ant。

假设我们希望在编译Snapshot版本的时候打开debug选项，否则关闭，在Ant中可以用Condition实现：

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

嗯，脚本有点长，和任务无关的东西有点多。

那么，如果改用Gradle来实现相同的任务会如何呢？

```gradle
apply plugin: 'java'

version = '1.0-SNAPSHOT'

compileJava {
  options.debug = version.endsWith('SNAPSHOT')
}
...
```

对比这两段脚本，我们就可以初步了解为什么我们需要新一代的构建工具：

* 程序员更容易读懂编程语言，而不是XML，后者擅长作为一种数据交换格式，是给机器读的（如Web Service）
* 编程语言的强大能力应为构建过程所用，如上面代码中的 `endsWith` 方法，而无需舍近求远地使用 `<contains>`
* 更重要的是，Java程序员无需翻阅文档就能使用 `if` `else` `endsWith` ，而 `<if>` `<else>` `<contains>` 则不见得
* 额外地，动态语言以及DSL能够使得程序员更专注在构建逻辑上，脚本也更为简洁，而在这方面XML真是望尘莫及

### Gradle是什么

* ant/maven/ivy/gradle都是些什么

### 语法简介

## Getting started
### 安装、环境

### 工程构建

* Java、Web 常用task及其惯例
* 大同小异 JVM语言 Groovy、Scala
* 其他语言的支持

### 多工程构建

### 依赖管理

* 引用
* 发布

## 插件

* built in/3'rd party
* 构建逻辑重用

### Gradle Wrapper

## 对比

* ant/maven/sbt

## 小结

## 学习资源？