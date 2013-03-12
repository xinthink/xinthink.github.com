# 拥抱Gradle: 下一代自动化工具

## 认识Gradle
过去Java世界的人谈起构建、自动化，Ant、Maven一定是必备的词汇吧，而如今，‘Gradle’这个名字也渐渐吸引了更多人的目光。今天我们就来认识一下这位号称‘下一代自动化工具’的Gradle。

不过，在此之前，我们先来温习一下既熟悉又陌生的Ant。

假设这里有一个使用Ant构建的Java工程，现在我们决定给Snapshot版本增加更多的debug信息，以下是一种可能的实现方式：

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

这段脚本当然能够工作，但我得承认，我无法在5秒钟之内看清楚它究竟表达了什么，其含义已被没完没了的标签所淹没。哦，你觉得挺好的，习惯了就好？那，如果这样的脚本有500行，甚至更长呢？

嗯，我想我们需要一个更好的解决方案，也许会是这样的：

```gradle
apply plugin: 'java'

version = '1.0-SNAPSHOT'

compileJava {
  options.debug = version.endsWith('SNAPSHOT')
}
```
这就是我们的第一段Gradle代码，如你所见，它要简短得多，而且你一定也找到了“熟面孔”：属性访问、赋值和方法调用。请注意我用了‘代码’一词，没错，Gradle脚本实际上就是一段Groovy代码。

对比这两段脚本，就能够对“我们为什么需要下一代构建工具”这个问题有了初步认识：

* 程序员更容易读懂编程语言编写的代码，而不是XML，后者擅长作为一种数据交换格式，是给机器读的（如Web Service）
* 编程语言的强大能力应为构建过程所用，如上面代码中的 `endsWith` 方法，而无需舍近求远地使用 `<contains>`
* 更重要的是，Java程序员无需翻阅文档就能使用 `if` ， `else` 和 `endsWith` ，而 `<if>` ， `<else>` 和 `<contains>` 则不见得
* 额外地，动态语言以及DSL能够使得程序员更专注在构建逻辑上，脚本也更为简洁，而在这方面XML真是望尘莫及

## Gradle是什么
在对Gradle有了一点感性认识以后，现在我们可以来讨论：Gradle究竟是什么？

讨论这个问题，我们还是得从Gradle的前辈们说起：Ant是一个非常通用的构建工具，对你做什么和怎么做几乎没有任何约束，但通用的代价是牺牲了特定领域的便捷性，于是Ivy为我们提供了统一管理外部依赖的机制，Maven则在同一个工具内提供了构建和依赖管理的能力，同时引入了惯例优先的理念，从而使得Java工程的构建变得便捷，也更规范，。

这些先辈都很棒，但也有着共同的弱点：它们都使用XML作为描述格式；它们的插件开发不是很便捷（XML与Java之间是存在鸿沟的）。

作为后起之秀，在吸收前辈们的精华并设法克服它们的弱点之后，就有了Gradle:

* 同时提供了构建和依赖管理的能力
* 完全支持Maven、Ivy的资源库（Repository），你只是换了一种使用方式，而不需要重建它们
* 但也不强制使用Maven、Ivy的XML描述符，你甚至可以使用一个普通的文件夹作为资源库，而仍可拥有依赖传递的特性（Transitive Dependency）
* 也提供了惯例优先模式，同时惯例默认值的覆盖变得更为容易
* 使用Groovy作为构建语言，并且提供了一套DSL，大大提升了构建（或自动化）过程的可编程能力和脚本的可读性，插件的编写、构建逻辑的复用也变得更为容易

## Gradle实战
现在，我们就通过具体的例子来进一步认识Gradle。虽然被设计为通用的构建工具，但无疑Java仍是Gradle的核心领域，我们就从这里开始。

### Java工程构建
#### 第一个例子
继续第一节中的例子，现在我们的Java工程构建已从Ant切换到了Gradle，可我们还没剖析过那段很酷的代码，它究竟是什么意思？

```gradle
apply plugin: 'java' // 引用Java插件

version = '1.0-SNAPSHOT' // 定义项目的版本号

// 覆盖compileJava任务（在Java插件中定义）默认的编译选项
compileJava {
  options.debug = version.endsWith('SNAPSHOT')
}
```
声明需要引用的插件，并覆盖惯例设置，就是这样！假设工程的目录结构符合惯例（与Maven的惯例相同）、不依赖第三方组件，那么它就已经可以正确地工作。

从这个简单的例子，我们可以了解一些Gradle的理念：

1. 在专注各个领域的插件（如Java）中封装构建所需的大部分（如果不是全部）工作，并提供合理的默认值，即惯例（如工程目录结构），保持开箱即用
2. 使用者的主要任务是根据实际情况覆盖各种默认设置

* 我们对Gradle的使用以及扩展，都应该符合这些理念，以便更好的享受Gradle带来的进步

#### 外部依赖
项目进展顺利，为了避免重新发明轮子，我们决定使用一些优秀的第三方组件。对外部依赖，遵循“一切皆为代码”的理念，声明第三方组件的标识符和准确的版本，而不是把200MB莫名其妙的jar文件一股脑的提交到SCM。

在build.gradle中的任意位置添加：

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

* 请仔细管理Configuration，这能让组件的构建和发布保持整洁，如果有必要你甚至可以定义自己的Configuration

#### 属性和环境变量
随着构建逻辑的增多，我们发现每次升级版本，都要在多处同时修改版本号。于是，我们把构建过程的一些全局性或者需反复引用的值抽取出来，统一在属性文件中定义，从而保持构建脚本的可维护性。

建议：
* 将工程范围内的全局属性放到工程（主工程或子工程均可）目录下的 `gradle.properties` 文件中
* 而与开发者个人相关且不便纳入版本控制的属性，放到 `$HOME/.gradle` 下的 `gradle.properties` 文件，比如签署app的密钥
* 与环境相关的属性（测试环境标志、工程发布目录等），则可通过 `-P` 参数在执行命令时传入，如 `gradle demo -Pdebug=line`（建议在CI环境中自动执行）

### 多工程构建
项目很成功，需求源源不断，现在我们的工程已经膨胀了数倍，考虑到并行开发和代码重用的需要，我们决定将原来的单个大工程拆分成多个较小的工程（模块）。

假设项目拆分成了 core 和 war 两个子工程，目录结构如下：

    demo                项目根目录
      |-- core          core 子工程
      |    |-- build    子工程的构建目录
      |    \-- src      子工程的源代码目录
      |
      \-- war           war 子工程
           |-- build    子工程的构建目录
           \-- src      子工程的源代码目录

在这样的多工程构建环境下，关键的问题包括：

1. 工程（模块）间的依赖关系，这关系到编译的先后顺序
2. 第三方依赖的统一管理，避免第三方组件的版本冲突（这是很让人头疼的问题）
3. 构建脚本的整洁，构建逻辑的重用

带着这些问题，我们开始在Gradle中使用多工程构建。首先需要声明子工程，在根目录下放置一个 `settings.gradle` 文件：

    include "core", "war"

在根工程的 `build.gradle` 脚本中定义公共的构建逻辑：

```gradle
subprojects {
  apply plugin: 'java'

  repositories {
    mavenCentral() 
  }

  // 所有的子项目都需要的依赖
  dependencies {
    compile 'org.slf4j:slf4j-api:1.6.6'
    ...
  }
}
```
subprojects 中定义的任何内容将所有子工程生效，你可以在这里定义属性、依赖，甚至task。

* 多项目环境下，执行task时，将会对所有适用的子工程进行调用，如： `gradle compileJava` 将编译所有的子工程
* 单独执行某个工程的task，需要指定工程前缀，如： `gradle :war:compileJava`

子工程如果没有特殊的需要，可以没有 `build.gradle` 文件，我们的 core 模块就是如此。不过很显然 war 模块需要成为一个 web 工程，而且需要引用 core 模块，因此，我们需要为 war 模块放置一个 `build.gradle`：

```gradle
apply plugin: 'war'
apply plugin: 'jetty'

dependencies {
  compile project(':core')
}
```
现在执行 `gradle :war:compileJava` ，Gradle将会确保 core 工程先被编译并打包。

好了，看来我们关注的多工程构建问题已经有了答案：

* 我们只需要声明子工程的依赖关系，Gradle将自动管理构建顺序，而这样的声明与第三方依赖的声明方式是一致的
* 公共的依赖统一声明，避免各自为政带来的混乱
* 在 `subprojects` `allprojects` 中定义公共的属性、逻辑和依赖，子工程只需进行增量定义或覆盖默认值即可

Gradle也采用多工程管理自身的源代码，对我们来讲这实在是再好不过的参考资源了。另外，Gradle也因此一定十分深刻地了解多工程构建的种种需求，从而进行更好的支持，Gradle也确实将多工程构建视为亮点之一。

### 依赖管理

* 引用
* 发布

### 插件

* built in/3'rd party
* 构建逻辑重用
为公司或者团队设计一套构建标准（惯例），并通过插件发布出来

### Gradle Wrapper

### 其他语言构建

## 对比

* ant/maven/sbt


## 小结
