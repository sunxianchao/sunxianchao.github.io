---
layout: post
title: "maven 多个模块聚合设置版本号"
description: ""
category:  maven
tags: [maven]
---

###问题描述
在使用maven多个模块聚合的时候，多个模块之间的version设置问题，一个项目作为一个整体，version这个重要的属性就应该保持一致！

###解决问题
方案一：在顶层的pom中设置一个globalVersion的全局变量，由于子模块的pom是一层层继承下来的，因此在子模块中也可以获取到这个变量，设置之后在本地，项目之间的依赖关系都正常，但是当操作mvn compile 等一系列生命周期的时候就出现download 一个 顶层pom:${globalVersion} 的pom文件，这里globalVersion变量无法识别。
<!--break-->

## 方案二：通过maven 的versions 插件实现，在打包之前在 parent pom中执行

mvn versions:set -DgenerateBackupPoms=false -DnewVersion=0.1
newVersion:是新的版本号 generateBackupPoms=false 这个如果不设置默认是true 会生产一个备份的pom文件

执行后会全部更改项目的pom中的version，这样就完美解决了。

pom中version的规范
所有的子模块中的pom不需要单独设置version 都可以忽略该属性，如果写了那么就需要用 ${project.parent.version}变量替换，同理在dependency 子模块的时候也是这样设置

