---
title: maven 学习
date: 2020-09-14 15:16:39
tags:
---

### maven 是为 java 项目打造的管理和构建工具，主要功能有:

- 提供了一套标准化的项目结构
- 提供了一套标准化的构建流程(编译，测试，打包，发布。。)
- 提供了一套依赖管理机制
- Maven 使用 pom.xml 定义项目内容，并使用预设的目录结构；
- 在 Maven 中声明一个依赖项可以自动下载并导入 classpath；
- Maven 使用 groupId，artifactId 和 version 唯一定位一个依赖
<!-- more -->
# maven 依赖关系

依赖关系
Maven 定义了几种依赖关系，分别是 compile、test、runtime 和 provided：

scope 说明 示例
compile 编译时需要用到该 jar 包（默认） commons-logging
test 编译 Test 时需要用到该 jar 包 junit
runtime 编译时不需要，但运行时需要用到 mysql
provided 编译时需要用到，但运行时由 JDK 或某个服务器提供 servlet-api

默认的 compile 是最常用的，Maven 会把这种类型的依赖直接放入 classpath
test 依赖表示仅在测试时使用，正常运行时并不需要。最常用的 test 依赖就是 JUnit：

```xml
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter-api</artifactId>
    <version>5.3.2</version>
    <scope>test</scope>
</dependency>
```

runtime 依赖表示编译时不需要，但运行时需要。最典型的 runtime 依赖是 JDBC 驱动，例如 MySQL 驱动：

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.48</version>
    <scope>runtime</scope>
</dependency>
```

provided 依赖表示编译时需要，但运行时不需要。最典型的 provided 依赖是 Servlet API，编译的时候需要，但是运行时，Servlet 服务器内置了相关的 jar，所以运行期不需要：

```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>4.0.0</version>
    <scope>provided</scope>
</dependency>
```

### 命令行编译

进入 `pom.xml` 目录，输入：

```bash
mvn clean package
```

这里注意，pom.xml 里面的 properties 指定的 source 和 target java version 要和项目的 java version 保持一致，在 idea 中，可以通过 `project structure` 指定下 java version
![](1.png)
![](2.png)
除此以外，配置 java 环境时候注意设置下 `CLASS_PATH`,`JAVA_HOME` 和 `M2_HOME`

```bash
CLASS_PATH= ;%JAVA_HOME%\lib;%JAVA_HOME%\lib\tools.jar
JAVA_HOME=C:\Program Files\Java\jdk1.8.0_261
M2_HOME=C:\software\apache-maven-3.6.3
```

```xml
<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<maven.compiler.source>8</maven.compiler.source>
		<maven.compiler.target>8</maven.compiler.target>
		<java.version>8</java.version>
</properties>
```

然后就可以在 `target` 目录获得编译后自动打包的 jar

我的博客即将同步至腾讯云+社区，邀请大家一同入驻：https://cloud.tencent.com/developer/support-plan?invite_code=3vwytile8e04o
