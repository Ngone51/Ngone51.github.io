---
layout:     post
title:      "Spark编译(使用阿里云maven仓库)"
subtitle:   "本文提供了在国内快速完成Spark编译的方法"
date:       2018-01-28 10:30:00
author:     "wuyi"
header-img: "img/post-bg-2015.jpg"
tags:
    - Spark
---

<p id = "build"></p>
---
我以近十天的编译Spark的痛苦经历，来分享一下如何在国内快速的完成Spark编译。

其实，唯一的任务就是将Spark中的默认maven中央仓库地址，替换成国内的maven仓库地址（我用的是阿里云的）。如果你能翻墙，那就不用浪费时间看下去了。

以下提及的spark均为2.11版本。

修改如下：

1. 采用maven编译spark：

```
build/mvn  -DskipTests clean package
```
那么，spark会默认下载apache-maven-3.3.9的maven版本来下载依赖包。
修改maven的配置文件spark/build/apache-maven-3.3.9/conf/settings.xml，添加阿里云仓库镜像地址：

```
<mirror>
  <id>central</id>
  <name>Nexus aliyun</name>
  <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
  <mirrorOf>central</mirrorOf>
</mirror>
```

2. 采用sbt编译spark：

```
build/sbt package
```

那么，spark会默认下载0.13.16的sbt版本(spark/project/build.properties中配置版本号)

并且，会在用户目录下，创建.sbt/目录。
进入该目录，创建repositories文件，添加以下内容：

```
[repositories]
local
central: http://maven.aliyun.com/nexus/content/groups/public/
typesafe: http://repo.typesafe.com/typesafe/ivy-releases/, [organization]/[module]/(scala_[scalaVersion]/)(sbt_[sbtVersion]/)[revision]/[type]s/[artifact](-[classifier]).[ext], bootOnly
sonatype-oss-releases
sonatype-oss-snapshots
```

**注意: maven地址对应的名称，一定要是central！！！否则，sbt依旧会使用默认的中央仓库的地址**

3. 修改IsolatedClientLoader#downloadVersion中maven仓库地址
打开IsolatedClientLoader.scala文件，找到downloadVersion()函数：

```
···
val classpath = quietly {
      SparkSubmitUtils.resolveMavenCoordinates(
        hiveArtifacts.mkString(","),
        SparkSubmitUtils.buildIvySettings(
          Some("http://www.datanucleus.org/downloads/maven2"),
          ivyPath),
        exclusions = version.exclusions)
    }
···
```

把some()里面的url地址替换为阿里云仓库地址。

这个问题在[SPARK-19836]以及pr#16803中有提出，但最终都没有给出好的解决方案。所以，这个问题遗留至今。

4. 修改SparkSubmit#createRepoResolvers的中maven仓库地址
打开SparkSubmit.scala文件，找到createRepoResolvers()函数：

```
// the biblio resolver resolves POM declared dependencies
    val br: IBiblioResolver = new IBiblioResolver
    br.setM2compatible(true)
    br.setUsepoms(true)
    br.setName("central")
    cr.add(br)

    val sp: IBiblioResolver = new IBiblioResolver
    sp.setM2compatible(true)
    sp.setUsepoms(true)
    sp.setRoot("http://dl.bintray.com/spark-packages/maven")
    sp.setName("spark-packages")
    cr.add(sp)
```
添加：

```
br.setRoot("http://maven.aliyun.com/nexus/content/groups/public/
")
```

以及，替换sp.setRoot()中的地址为阿里云的仓库地址。
*在build test的过程，VersionSuite会用到这几个地址。在我使用阿里云的地址之前，曾N多次一整个晚上阻塞在这个Suite上，最终放弃。*

5.修改SparkBuild#L218的maven仓库地址

```
// Override SBT's default resolvers:
    resolvers := Seq(
      DefaultMavenRepository,
      Resolver.mavenLocal,
      Resolver.file("local", file(Path.userHome.absolutePath + "/.ivy2/local"))(Resolver.ivyStylePatterns)
    ),
```

替换DefaultMavenRepository为：

```
Resolver.url("central", url("http://maven.aliyun.com/nexus/content/groups/public/")),
```

**同样的，url的名称必须为central**


备注：
1. 在上述修改完成之后，在dev/run-tests执行过程中，到了'Detecting binary incompatibilities with MiMa'的步骤，可能还有问题，例如：

```
[error] (mllib-local/*:mimaPreviousClassfiles) sbt.ResolveException: unresolved dependency: org.scala-lang#scala-reflect;2.11.4: not found
[error] (mllib/*:mimaPreviousClassfiles) sbt.ResolveException: unresolved dependency: org.scala-lang#scalap;2.11.0: not found
[error] (streaming-kafka-0-10/*:mimaPreviousClassfiles) sbt.ResolveException: unresolved dependency: log4j#log4j;1.2.15: not found
[error] (core/*:mimaPreviousClassfiles) sbt.ResolveException: unresolved dependency: org.scala-lang#scalap;2.11.0: not found
[error] (sql/*:mimaPreviousClassfiles) sbt.ResolveException: unresolved dependency: org.scala-lang#scalap;2.11.0: not found
[error] (streaming/*:mimaPreviousClassfiles) sbt.ResolveException: unresolved dependency: org.scala-lang#scalap;2.11.0: not found
[error] (graphx/*:mimaPreviousClassfiles) sbt.ResolveException: unresolved dependency: org.scala-lang#scalap;2.11.0: not found
[error] Total time: 114 s, completed 2018-1-7 16:02:17
[error] running /Users/wuyi/workspace/spark/dev/mima -Phadoop-2.6 -Pkubernetes -Phive-thriftserver -Pflume -Pkinesis-asl -Pyarn -Pkafka-0-8 -Phive -Pmesos ; received return code 1
```

解决办法：手动的下载对应jar包的pom文件，放到.m2/repositroy本地仓库中。

这个可能与第5步的修改有关，目前还不清楚。

2. 阿里云maven仓库有可能出现没有对应jar包的情况，此时也需要手动把jar包拷贝到本地仓库中。

3. 如果你需要向spark提交pr，那么，请记得chekout -- 刚刚被修改的几个源码文件。


