---
layout: post
title: "Tomcat 远程部署"
description: "通过 Ant 实现 Tomcat 的远程部署"
date: 2017-06-06
tags: [ant, tomcat]
comments: true
share: true
---

> 还是上次提到到公司的那个很坑的系统，然后我又负责系统的自动化部署，因此遇上了这个坑。这系统的测试服务器是台Windows的，而 jenkins 装在一台 linux 的服务器上面，本人 shell 脚本能力较弱，无法使用 shell 脚本与 Windows 服务器进行交互，因此使用了这个办法。步骤如下。  

### 环境概述

	* JDK 1.7
	* Ant 1.8.4
	* Tomcat 7

### 添加 Tomcat 用户
修改 `$CATALINA_HOME/conf/tomcat-users.xml`

```xml
<tomcat-users>
	<role rolename="manager-gui"/>
	<role rolename="manager-script"/>
	<user username="tomcat" password="tomcat" roles="manager-gui, manager-script" />
</tomcat-users>
```
然后访问该 tomcat 服务器，点击 manager app 按钮输入刚刚配置的 username 和 password ，如果登陆成功的说明配置完毕已经。页面也下图：

![tomcat index](/images/2017-06-06-tomcat-index.png)

![tomcat manager app](/images/2017-06-06-tomcat-manager-app.png)

### 复制 Jar 包
将 tomcat 服务器下的 jar 复制到本地 `$ANT_HOME/lib` 下，jar 包列表如下：

	* $CATALINA_HOME_lib_catalina-ant.jar
	* $CATALINA_HOME_lib_tomcat-coyote.jar
	* $CATALINA_HOME_lib_tomcat-util.jar
	* $CATALINA_HOME_bin_tomcat-juli.jar

### 修改build.xml

```xml
<property name="url" value="http://ip:port/manager/text"/>
<property name="username" value="tomcat"/>
<property name="password" value="tomcat"/>
<property name="contextPath" value="/"/>

<target name="_def_tomcat_tasks">
        <taskdef name="deploy" classname="org.apache.catalina.ant.DeployTask"/>
</target>

 <target name="deploy" description="Install web application" depends="compile,_def_tomcat_tasks">
        <deploy url="${url}" username="${username}" password="${password}" path="${contextPath}" war="${basedir}/${warFileName}" update="true" />
</target>
```

### 执行命令，进行远程部署

```bash
ant deploy
```
