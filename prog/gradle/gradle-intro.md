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
* 更重要的是，Java程序员无需翻阅文档就能使用 `if` ， `else` 和 `endsWith` ，而 `<condition>` 和 `<contains>` 则不见得
* 额外地，动态语言以及DSL能够使得程序员更专注在构建逻辑上，脚本也更为简洁，而在这方面XML真是望尘莫及

## Gradle是什么
在对Gradle有了一点感性认识以后，现在我们可以来讨论：Gradle究竟是什么？

讨论这个问题，我们还是得从Gradle的前辈们说起：Ant是一个非常通用的构建工具，对你做什么和怎么做几乎没有任何约束，但通用的代价是牺牲了特定领域的便捷性，于是Ivy为我们提供了统一管理外部依赖的机制，Maven则在同一个工具内提供了构建和依赖管理的能力，同时引入了惯例优先的理念，从而使得Java工程的构建变得便捷，也更规范。

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
声明需要引用的插件，并覆盖惯例设置，就是这样！假设工程的目录结构符合惯例（与Maven相同）、不依赖第三方组件，那么它就已经可以正确地工作。

从这个简单的例子，我们可以了解一些Gradle的理念：

1. 在专注各个领域的插件（如Java）中封装构建所需的大部分（如果不是全部）工作，并提供合理的默认值，即惯例（如工程目录结构），保持开箱即用
2. 使用者的主要任务是根据实际情况覆盖各种默认设置

* 我们对Gradle的使用以及扩展，都应该符合这些理念，以便更好的享受Gradle带来的进步

#### 依赖管理
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

与Maven一样，Gradle默认会自动解析、下载间接的依赖（即Transitive Dependency Management），大多数情况下这个特性很方便，但总有例外的时候，当使用的第三方组件增多，间接依赖的组件就容易出现冲突的情况，这个时候，一个Gradle命令可以帮助我们， `gradle dependencies` ，可以打印出工程的依赖树，而且这个命令不需要额外的插件就可以使用。如果存在冲突的依赖，可以关闭依赖传递或使用 `exclude` 排除不合适的依赖，再显式声明我们认为正确的依赖，例子：

```gradle
// 排除全部或特定的间接依赖
runtime ('commons-dbcp:commons-dbcp:1.4') {
  transitive = false
  // 或 exclude group: 'commons-pool', module: 'commons-pool'
}

// 然后显式声明
runtime 'commons-pool:commons-pool:1.6'
```

* 小心管理工程的依赖，必要时需排除冲突的间接依赖
* 题外话， `gradle dependencies` 命令容易敲错？可以使用缩写 `gradle dep` ，不必输入Task的全名，敲入字母（或驼峰首字母缩写）直至能够唯一识别即可

#### 属性和环境变量
随着构建逻辑的增多，我们发现每次升级版本，都要在多处同时修改版本号。于是，我们把构建过程的一些全局性或者需反复引用的值抽取出来，统一在属性文件中定义，从而保持构建脚本的可维护性。

建议：
* 将工程范围内的全局属性放到工程（主工程或子工程均可）目录下的 `gradle.properties` 文件中
* 而与开发者个人相关且不便纳入版本控制的属性，放到 `$HOME/.gradle` 下的 `gradle.properties` 文件，比如签署app的密钥
* 与环境相关的属性（测试环境标志、工程发布目录等），则可通过 `-P` 参数在执行命令时传入，如 `gradle demo -Pdebug=line`（建议在CI环境中自动执行）

### 多工程构建
项目很成功，需求源源不断，现在我们的工程已经膨胀了数倍，考虑到并行开发和代码重用的需要，我们决定将原来的单个大工程拆分成多个较小的工程（模块）。

假设项目拆分成了core和web两个子工程，目录结构如下：

    demo                项目根目录
      |-- core         core子工程
      |    |-- build    子工程的构建目录
      |    \-- src      子工程的源代码目录
      |
      \-- web          web子工程
           |-- build    子工程的构建目录
           \-- src      子工程的源代码目录

在这样的多工程构建环境下，关键的问题包括：

1. 工程（模块）间的依赖关系，这关系到编译的先后顺序
2. 第三方依赖的统一管理，避免第三方组件的版本冲突（这是很让人头疼的问题）
3. 构建脚本的整洁，构建逻辑的重用

带着这些问题，我们开始在Gradle中使用多工程构建。

首先需要声明子工程，在根目录下放置一个 `settings.gradle` 文件：

    include "core", "web"

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
subprojects 中定义的任何内容将对所有子工程生效，你可以在这里定义属性、依赖，甚至Task。

* 多项目环境下执行Task时，将会对所有适用的子工程进行调用，如： `gradle compileJava` 将编译所有的子工程
* 单独执行某个工程的Task，需要指定工程前缀，如： `gradle :core:compileJava`

子工程如果没有特别的需要，可以没有 `build.gradle` 文件，我们的core模块就是如此。不过很显然web模块应该是一个JEE Web应用，而且需要引用core模块，因此，我们为它添加一份 `build.gradle`：

```gradle
apply plugin: 'war'
apply plugin: 'jetty'

dependencies {
  compile project(':core')
  providedCompile 'javax.servlet:javax.servlet-api:3.1'
}
```

* `providedCompile` （在war插件中定义）可以确保servlet-api能够在编译时被引用，却不随web工程发布（运行时由Web容器提供）

现在执行 `gradle :web:compileJava` ，Gradle将会确保core工程先被编译并打包；执行 `gradle :web:assemble` 得到的war包也将包含core.jar。

看来我们关注的多工程构建问题已经有了答案：

* 我们只需要声明子工程间的依赖关系，Gradle将自动管理构建顺序，而这样的声明与第三方依赖的声明方式是一致的
* 公共的依赖统一声明，避免各自为政带来的混乱
* 在 `subprojects` `allprojects` 中定义公共的属性、逻辑和依赖，子工程只需进行增量定义或覆盖默认值即可

Gradle也采用多工程管理自身的源代码，因此一定十分深刻地了解多工程构建的种种需求，从而进行更好的支持，Gradle也确实将多工程构建视为其亮点之一。

### 发布组件
随着企业规模的扩大，我们开始需要在团队之间共享组件，这时企业内部的Maven资源库镜像就派上了用场（在我看来，资源库无疑是Maven最成功之处）。

我们的Core组件是如此酷，以至于其他团队天天嚷着要引入，好吧，现在让我们看看如何将组件发布到Maven资源库。

首先定义发布的目标资源库，现在core工程也需要 `build.gradle` 了：

```gradle
apply plugin: 'maven'

uploadArchives {
  repositories {
    mavenDeployer {
      repository(url: <repo_url>) {
        authentication(userName: <repo_user>, password: <repo_passwd>)
      }
    }
  }
}
```
这是给 `uploadArchives` 任务添加了一个目标资源库，其中 `<repo_url> <repo_user> <repo_passwd>` 分别为目标资源库的位置和认证信息（还记得吗？身份认证信息最好放在 `$HOME/.gradle/gradle.properties` 中，脚本中通过属性引用）

既然使用Maven资源库，最好还是按Maven的惯例，补充完整组件描述符（POM）：

```gradle
apply plugin: 'maven'

uploadArchives {
  repositories {
    mavenDeployer {
      repository(url: <repo_url>) {
        authentication(userName: <repo_user>, password: <repo_passwd>)
      }
      pom.project {
        name 'core'
        description '<project_desc>'
        url '<project_url>'
        artifactId 'core'
        packaging 'jar'
        scm {
          ...
        }
        developers {
          ...
        }
        ...
      }
    }
  }
}
```

看看最后这一串大括号，是否觉得嵌套层次有点深？让我们稍稍整理一下：

```gradle
apply plugin: 'maven'

ext.pomCfg = {
  name 'core'
  ...
}

uploadArchives.repositories.mavenDeployer {
  repository(url: '<repo_url>') {
    authentication(userName: '<repo_user>', password: '<repo_passwd>')
  }
  pom.project pomCfg
}
```
好了，这样看着就舒服多了。而且，现在我们的Core组件已经发布到了内部的Maven资源库，可以供其他团队引用了。

### 定义企业内部的构建规范
一个企业或团队，在运作过程中或多或少一定会积累下来一些约定（或称为规范、最佳实践），或许我们会有一个独特的工程目录结构，或者经过调优的编译选项，等等。企业如果希望提升构建的自动化程度，就应该考虑把这些规范固化下来，并在多个团队中共享。

由于Gradle构建是基于惯例的，重定义惯例也更为容易，这就使得它天然地成为企业构建规范定义和发布的最佳工具。在Gradle中，构建规范可以通过插件的形式定义和发布。

而Gradle编写插件的方式有很多种：

* 共享脚本 - 即把可重用的gradle片段摘取到一个独立的文件，然后通过 `apply from` 的方式引用，这是最廉价的一种方式
* buildSrc - 执行时，Gradle会自动引用buildSrc目录下的插件代码，它可以是构建脚本的片段，也可以是与独立插件一样的Groovy、Java代码
* 独立插件 - 扩展Gradle功能的插件，使用Groovy、Java编写，通过jar包发布。

由于我们主要考虑的是共享构建规范，因此第一种方式比较合适。

我们把约定的工程目录、编译选项等单独定义至 `common.gradle` ：

```gradle
apply plugin: 'java'

// 编译器选项
sourceCompatibility = 1.6
targetCompatibility = 1.6

tasks.withType(Compile) {
  options.encoding = 'utf-8'
}

// 源代码目录结构
sourceSets {
  main {
    java.srcDirs (
      'src/domain',
      'src/service',
      'src/controller'
    )
  }
}

task mkSrcDirs {
  description = 'Create all source dirs'
  doLast {
    sourceSets*.java.srcDirs*.each { it.mkdirs() }
    sourceSets*.resources.srcDirs*.each { it.mkdirs() }
  }
}

// 企业内部资源库
repositories {
  mavenRepo url: <company_public_repo>
  mavenRepo url: <company_private_repo>
}
```

我们在内网的站点共享了这个文件，供各个项目引用：

```gradle
apply from: <link_to_common_gradle>

// 项目特定的构建逻辑
...
```

### Gradle Wrapper
随着时间的推移，Gradle版本也在不断升级，有些前卫的家伙总是把Gradle更新到最新版本，而菜鸟们的机器上根本就没有Gradle，这就使得我们的工程构建可能在不同环境下出现不同的结果。尤其当我们需要在多台服务器上同时进行构建、发布的时候，这个问题就更为突出了。

这个问题早在Ant、Maven的时代就存在了，不过现在终于有了更好的解决方案：Gradle Wrapper。我们这就试试，实际上只需要这么一个Task：

```gradle
task wrap(type: Wrapper) {
  gradleVersion = '1.4' // 声明版本
  scriptFile = 'g' // 默认gradlew，太长
}
```
只需一人（通常为工程的创建者）执行此Task，生成 `g` 和 `g.bat` 脚本及支持性的jar包，把这些内容和工程文件一起放入SCM。其他开发者取得后，立刻可以通过 `g` 或 `g.bat` 脚本执行Gradle命令，不需要预先安装Gradle运行环境，第一次执行命令时会自动下载、安装。

* 遗憾的是，由于Gradle包的Size以及网络位置的问题，等待下载时要有耐心，建议在内网预先缓存该二进制包

这大概也可以看作”一切皆为代码“理念的进一步延续，即，我们声明需要使用的Gradle版本，使用时则会自动下载，确保在任何机器上都能够还原期望的构建环境。

## 小结
在本文中，我们通过模拟一个Java项目从无到有、从小到大的发展过程，逐步展现了Gradle为构建工作（或更广泛地：自动化工作）带来的变革，它所展现的‘力与美’让人惊艳，尤其对于Java社区那些一本正经的OOer们，他们就只有O_o的份了（玩笑，无贬义）。

从中我们可以一窥构建工具的发展趋势：

* 对构建语言的选用，真正的编程语言（及基于此的DSL）取代XML。严格来讲Ant、Maven的Schema也是一种DSL，但XML表达能力的缺陷最终使它们败下阵来
* 构建语言与工程语言一致或相近，如Ruby的Rake，JVM语言的Gradle、SBT，使得程序员更容易上手，同时他们的编程语言知识也得以在构建工作中一显身手
* 基于惯例的构建，使我们不必一再重复一些‘显而易见’的配置，我们还可以更容易地规范企业内部的构建过程，从而进一步提升企业的效率
* 一切皆代码，组件依赖、构建环境等都成为可以进入SCM的‘代码’，从而无论在何处，都可以还原出我们所期望的构建环境

Gradle还刚刚起步，却已经吸引了Spring Framework、Hibernate、Grails等大名鼎鼎的用户，有理由相信，Grails未来的版本还会带给我们更大的惊喜。
