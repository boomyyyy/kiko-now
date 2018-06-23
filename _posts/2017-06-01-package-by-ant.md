---
layout: post
title: "使用ant对普通web项目进行打包"
description: "ant 打包普通 web项目"
date: 2017-6-01
tags: [ant, 工具]
comments: true
share: true
---
build.xml 内容如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- 定义一个工程，默认任务为warFile。 -->
<project name="elite" default="warFile" basedir="/Users/zhangyong/code/work/elite">

    <!-- 定义属性，打成war包的名称。 -->
    <property name="warFileName" value="elite.war"></property>

    <!-- 定义路径，编译java文件时用到的jar包。 -->
    <path id="project.lib">
        <fileset dir="${basedir}/WebRoot/WEB-INF/lib">
            <include name="**/*.jar"/>
        </fileset>
    </path>

    <!-- 定义任务，清空任务：清空原有的class文件，创建新的build路径。 -->
    <target name="clean">
        <delete dir="${basedir}/build" />
        <mkdir dir="${basedir}/build" />
    </target>

    <!-- 定义任务，编译src文件夹中的java文件，编译后的class文件放到创建的文件夹下。 -->
    <target name="compile" depends="clean">
        <javac srcdir="${basedir}/src" destdir="${basedir}/build" includeantruntime="false">
            <classpath refid="project.lib">
            </classpath>
        </javac>
    </target>

    <!-- 定义默认任务，将class文件集合成jar包。 -->
    <target name="warFile" depends="compile">
        <!-- 删除原有war包。 -->
        <delete dir="${basedir}/${warFileName}" />
        <!-- 建立新war包。 -->
        <war destfile="${basedir}/${warFileName}" webxml="${basedir}/WebRoot/WEB-INF/web.xml">
            <!-- 将非jar和非class文件拷贝到war包的对应路径下。 -->
            <fileset dir="${basedir}/WebRoot">
                <include name="**/**.*" />
                <exclude name="**/*.jar"/>
                <exclude name="**/*.class"/>
            </fileset>
            <!-- 将jar和class文件拷贝到war包的对应路径下。 -->
            <lib dir="${basedir}/WebRoot/WEB-INF/lib" />
            <classes dir="${basedir}/build" />
        </war>
    </target>

</project>

```
