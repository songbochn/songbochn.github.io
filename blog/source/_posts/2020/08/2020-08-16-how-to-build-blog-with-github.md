---
title: 博客迁移到gitpages上(一)
tags:
  - hexo
  - gitpages
categories:
  - 写作
date: 2020-08-16 17:29:58
---
#### 前言

最近在思考工作这几年收获的时候，发现自己已经失去了写文章的好习惯，究其原因也还是不够重视，常常感叹自己的碎片化思考过多，难以形成知识体系，导致有些事情常常被反复提起，却未曾进行实践，占据这大部分记忆的空间。也希望通过博客的方式将这种思考记录下来，并且通过写作反过来也能够帮助自己想清楚问题。帮助自己养成习惯。

#### 为何将迁移到gitpages

之前觉得自己购买vps搭建博客很酷，也就操作了一波。但在此之后更新博客的频率也少了起来，究其原因是什么呢？我觉得是成本在作祟，当我们做一件事情的成本够低，才能够养成习惯。玩游戏，刷抖音都是如此。因此这次犯了一个严重的错误，增加了自己更新博客的成本，但其实也熟悉了背后的全链路流程，也算有些收获。第二点是统一入口很重要，如果能够将所有相关的操作都集中在一个入口，那就会减少记忆和切换的时间。计划后续将博客迁移到github上，一方面在逛github的时候遇到好玩的东西就会去研究，顺手就能整理成博文，将其切换的成本降到最低。
<!--more-->

#### 如何迁移

##### 注册github

使用gmail邮箱就可以注册github网站。

##### 搭建一个demo
具体过程可以参考

>[gitpages官方文档](https://pages.github.com/)

简单的说，第一步在github上创建一个仓库，仓库名为`username,github.io`，将username替换成你的用户名。然后将该仓库clone下来，添加一个index.html,在访问https://username.github.io后就会看到index.html里的内容了。

##### 使用hexo为博客添加样式

>[hexo官方文档](https://hexo.io/zh-cn/)

因为我之前的博客使用Hexo编译静态文件的，所以后续采用Hexo来管理。
Hexo是基于`Node.js`的组件，需要先安装`Node.js`。[下载地址](https://nodejs.org/en/download/) 根据自己的操作系统下载对应版本安装即可。
在安装完成之后执行以下命令
```
npm install -g hexo-cli
```
hexo就安装完成。
在安装完成之后，切换到刚才clone的目录下，执行以下命令
```
hexo init blog
cd blog
npm install
hexo generate
hexo server
```
此时通过http://localhost:4000/就可以访问到你的网站首页了。
##### 部署到git上
> 参考[一键部署](https://hexo.io/zh-cn/docs/one-command-deployment)

执行以下命令:
```
npm install hexo-deployer-git --save
```
同时修改_config.yml里的参数
```
deploy:
  type: git
  repo: https://github.com/username/username.github.io
  branch: gh-pages
  message: '站点更新: {{ now("YYYY-MM-DD HH:mm:ss") }})'
```
这里的分支写的是gh-pages，我们会渲染之后的结果推到这个分支上，在master或者新建的分支上进行源代码的管理，我这里采用新建的feature/blog分支进行源代码管理。
如果我们将渲染结果推到gh-pages上，需要在github上对应rep的setting->options->gitpages那一栏把source branch变更为gh-pages这个分支，save就可以了。
在此之后我们通过hexo deploy就可以把源代码进行编译并且结果推到github上，网页也会实时刷新了。

##### 开始写作
我们无时无刻要追求者效率。每次在写文章的时候需要写一些通用的内容，包含日期，时间，分类，便签等等信息，而我们每次在执行hexo new "文章标题"的时候其实使用到了一些模块，这些模板放在scaffolds的文件夹里，包含post.md,page.md,draft.md三类模板，关于这三类的区别可以参考官方文档和附录2。我们也可以自定义模板，在执行命令`hexo new 模板名 文章标题`来使用我们的模板。我们自定义的模板都会放在post文件夹，我们写文章常用的是post模板。
下面为文章标题加上日期。
在_config.yml里的new_post_name改为:year/:month/:day/:title.md
现在我们通过以下代码加上分类和标签。编辑post.md文件
```
---$
title: {{ title }}$
date: {{ date }}$
tags:$
 - $
categories: $
 - $
---$
```
接下来执行`hexo new 文章标题`就可以了,但是我们在执行之后还要去对应的目录找，然后再用编辑器打开，那么能不能做到执行完就打开呢？当然是可以的。需要新建一个js脚本
找到hexo目录下的scripts目录(没有则新建一个)
新建openAfterNew.js文件(名称可以随便起)
填写以下内容
```
var exec = require('child_process').exec;
hexo.on('new', function(data){
  exec('open -a "/Applications/Sublime Text.app" ' +data.path);
});
```
然后保存，执行`hexo new 文章标题`就可以直接开始写作了~

##### 彩蛋

如果我们有多个网站需要管理怎么办呢？现在username.github.io这个域名不是被占用了吗？其实这个时候我们可以新建一个repo或者使用已存在的repo，在setting->options->gitpage那一栏选择使用的分支，这样就可以通过username.github.io/reponame来访问了，reponame替换为对应的仓库名。

##### 参考资料
1. [Github Pages部署个人博客（Hexo篇）](https://juejin.im/post/6844903590369181703)
2. [[hexo]如何生成一篇新的post](https://oakland.github.io/2016/05/02/hexo-%E5%A6%82%E4%BD%95%E7%94%9F%E6%88%90%E4%B8%80%E7%AF%87%E6%96%B0%E7%9A%84post/)
3. [在 hexo new 之后立即打开新建的 Markdown 文稿](https://liam.page/2015/05/01/open-editor-after-hexo-new-immediately/)