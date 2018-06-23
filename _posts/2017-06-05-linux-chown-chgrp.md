---
layout: post
title: linux 修改文件所属用户和组
description: 使用 chown 和 chgrp 切换文件或目录所属用户和组 
date: 2017-06-05
tags: [linux] 
comments: true
share: true
---
## 使用chown命令可以修改文件或目录所属的用户：

* 命令：chown 用户 目录或文件名
* 例如：chown qq /home/qq  (把home目录下的qq目录的拥有者改为qq用户) 

## 使用chgrp命令可以修改文件或目录所属的组：

* 命令：chgrp 组 目录或文件名
* 例如：chgrp qq /home/qq  (把home目录下的qq目录的所属组改为qq组)
