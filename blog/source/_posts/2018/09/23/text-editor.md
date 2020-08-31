---
title: 用sublime写markdown，多款编辑器使用后的归属
date: 2018-09-23 02:43:12
toc: true
tag:
    - 编辑器
    - 效率工具
categories:
    - [效率工具,编辑器]
    - [文本,编辑]
---

#### 前言

在平常的文字编辑以及生活中，一方面需要记录自己的想法灵感，一方面需要写文章博客。因此一个好且顺手的编辑器是非常重要的。

之前用备忘录来做平时记录，以及在写markdown类型的博客的时候使用macdown，但是在编辑表格时候总是很难受。google过很多的的编辑器，对markdown的
<!--more-->
表格编辑都支持的不是很好，于是试试sublime。后边发现有很好用的插件支持，就它了！

#### 插件
在用sublime写markdown要支持一下几个功能,markdown语法高亮，预览以及好用的表格编辑插件

- Markdown Editing
该插件主要是作为一个语法高亮的功能存在，这是[github地址](https://github.com/SublimeText-Markdown/MarkdownEditing),安装过程在github里都有，这里要注意首先安装sublime的包管理工具，安装过程点[这里](https://packagecontrol.io/installation)

- Markdown Preview 关于预览插件，有一个MarkdownLivePreview插件可以做到实时预览但是效果不好，因此推荐Markdown Preview插件。虽然做不到实时预览，但是可以让你花更多的时间在内容还不是格式上。这是[github地址](https://facelessuser.github.io/MarkdownPreview/install/),如果安装了上述的包管理工具，一切都很快。关于预览快捷键的设定。
```bash
Preference -> Key bindings -> 然后在右侧输入
{ "keys": ["alt+m"], "command": "markdown_preview", "args": {"target": "browser", "parser":"markdown"} }
```
然后再按alt+m都可以在浏览器打开了

- Table Editor 这个插件可以说是深深解决了我等编写markdown表格的痛点，这是[github地址](https://github.com/vkocubinsky/SublimeTableEditor),安装完基本上一个tab键就可以解决问题了，具体用法参考官方文档介绍。

#### FAQ

##### 在写markdown的时候如果文本为居中，此时希望移到左侧
```bash
Preference -> Package Settings -> Markdown Editing -> 选择你相应主题的user文件
写入 "draw_centered": false, // 改为false，原始值为true
```
此时就不会出现左侧大量留白的现象了。