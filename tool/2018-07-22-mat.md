---
title: MAT工具
date: 2018-07-22 20:00:23
tag: 
	- 内存分析
	- 工具
category: 
	- jvm常用工具
---

#### 安装

首先现在官网下载MAT

将mat.app移动到/Application

修改MemoryAnalyze.ini

添加目标地址 （写在-vagx前）

-data 

/地址

如果打开UI没反应，，需要下载swt包并更换

<!--more-->

如果显示修改java heap堆得情况

进入MemoryAnalyze.ini修改参数

#### 我的配置

```
-startup
../Eclipse/plugins/org.eclipse.equinox.launcher_1.3.100.v20150511-1540.jar
--launcher.library
../Eclipse/plugins/org.eclipse.equinox.launcher.cocoa.macosx.x86_64_1.1.300.v20150602-1417
-keep_unreachable_objects
-data
/Users/songbo/work/tool/workspace
-vmargs
-Xmx4g
-Dorg.eclipse.swt.internal.carbon.smallFonts
-XstartOnFirstThread
```

如果你打开MAT看使用空间大小，会发现远小于jmap看的使用情况，这是因为mat只保留了GCroot能找到的对象，这个时候需要打开偏好设置，勾上keep unreachable objects选项。

#### 参考文档

[内存分析插件 MAT](https://www.cnblogs.com/kongzhongqijing/articles/3625867.html)

[How to analyse unreachable objects](https://wiki.eclipse.org/MemoryAnalyzer/FAQ#How_to_analyse_unreachable_objects)

[mat 使用笔记](http://inter12.iteye.com/blog/1407492)