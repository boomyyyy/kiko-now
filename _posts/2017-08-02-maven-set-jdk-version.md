---
layout: post
title: "maven 配置 jdk 版本"
description: "maven 配置 jdk 版本"
date: 2017-08-02
tags: [Java, 工具, maven, 备忘随手记]
comments: true
share: true
---

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <configuration>
        <source>1.8</source>
        <target>1.8</target>
      </configuration>
    </plugin>
  </plugins>
</build>
```
