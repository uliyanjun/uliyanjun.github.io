---
title: Maven Scope 详解
date: 2024-12-04 13:45:43
categories:
  - java
tags:
  - maven
---

# Scope 类型

## compile

- 默认依赖范围，是一个比较强的依赖，范围：编译、测试、运行。如 `spring-boot`。

```xml
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot</artifactId>
    <version>3.3.2</version>
    <scope>compile</scope>
  </dependency>
```



## test

- 仅测试有效，如 `junit`。

```xml
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>3.8.1</version>
    <scope>test</scope>
  </dependency>
```



## provided

- 编译和测试时需要，如 `javax.servlet-api`，因为运行时，`Servlet` 容器已经提供了。

```xml
  <dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>4.0.1</version>
    <scope>provided</scope>
  </dependency>
```



## runtime

- 运行和测试的时候需要，如 `mysql-connector-java`。

```xml
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>8.0.33</version>
  <scope>runtime</scope>
</dependency>
```



## system

- 生命周期同 `provided`，但必须显式指定位于本地系统中 `jar` 文件的路径。

```xml
<dependency>
  <groupId>your.group</groupId>
  <artifactId>your-artifact-id</artifactId>
  <version>1.0.0</version>
  <type>jar</type>
  <scope>system</scope>
  <systemPath>${project.basedir}/libs/your-artifact-id-1.0.0.jar</systemPath>
</dependency>
```



## import

- 引入一个 `pom`  文件的 `<dependencys>`。
- `scope` 为 `import` 的这个 `<dependency>` 会被替换为引入的 `<dependencys>`。
- `import` 只能出现在打包方式为 `pom` 的工程中。
- `import` 只能出现在 `dependencyManagement` 节点下。



Project A:

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>maven</groupId>
  <artifactId>A</artifactId>
  <packaging>jar</packaging>
  <name>A</name>
  <version>1.0</version>
  <dependencies>
    <dependency>
      <groupId>group-a</groupId>
      <artifactId>artifact-a</artifactId>
      <version>1.0</version>
    </dependency>
    <dependency>
      <groupId>group-a</groupId>
      <artifactId>artifact-b</artifactId>
      <version>1.0</version>
      <scope>runtime</scope>
    </dependency>
  </dependencies>
</project>
```

Project B:

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>maven</groupId>
  <artifactId>B</artifactId>
  <packaging>pom</packaging>
  <name>B</name>
  <version>1.0</version>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>maven</groupId>
        <artifactId>A</artifactId>
        <version>1.0</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>test</groupId>
      <artifactId>a</artifactId>
      <version>1.0</version>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>test</groupId>
      <artifactId>c</artifactId>
      <scope>runtime</scope>
    </dependency>
  </dependencies>
</project>
```

Project B 效果：

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>maven</groupId>
  <artifactId>B</artifactId>
  <packaging>pom</packaging>
  <name>B</name>
  <version>1.0</version>
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>group-a</groupId>
        <artifactId>artifact-a</artifactId>
        <version>1.0</version>
      </dependency>
      <dependency>
        <groupId>group-a</groupId>
        <artifactId>artifact-b</artifactId>
        <version>1.0</version>
        <scope>runtime</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>test</groupId>
      <artifactId>a</artifactId>
      <version>1.0</version>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>test</groupId>
      <artifactId>c</artifactId>
      <scope>runtime</scope>
    </dependency>
  </dependencies>
</project>
```



# 依赖传递

现在有项目 A、B、C，其中 A 依赖 B，B 依赖 C。

- 当 C 中依赖 `scope` 为 `test` 或者 `provided` 时，A 不依赖 C。
- 否则，A 依赖 C，且 `scope` 与 B 相同。



# 参考资料

- https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html#Dependency_Scope
- https://open.alipay.com/portal/forum/post/138401042
- https://blog.51cto.com/knifeedge/5273943