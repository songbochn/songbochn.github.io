---
title: 博客迁移到gitpages上(二)
tags:
  - gitpages
  - domain
  - 域名
categories:
  - 博客
date: 2017-10-05 12:58:29
---

#### 前言

接上次博客能够正常跑起来并且发表文章之后，我们可以对博客做进一步的功能增强。加入一些我们的自定义配置。

#### 自定义配置

##### 自定义域名

首先我们做的就是把默认的username.github.io域名进行更新,变成我们自定义的域名。这里可以去[namecheap](https://www.namecheap.com/)进行购买，我选的域名的考虑点是这个网站的特点，这个网站希望传达的理念是什么？，最终定义为知行，也是mit的校训:mind and hand,但是发现已经被注册过，就选择了英语和拉丁文拼接，形成了现在的域名:mindmanus.com
<!--more-->

在购买域名之后下一步就是域名解析，目的通过该域名解析到username.github.io上。在这里我们选择dnspod作为我们的dns解析服务提供商。首先需要在namecheap上将域名服务器(nameservers)配置为dnspod的服务器，一般来说为f1g1ns1.dnspod.net和f1g1ns2.dnspod.net。在此之后namecheap的配置工作就做完了，后续的映射包括子域花费【类似blog.manusmens.com】都是在域名服务器【dnspod】上进行配置的。

其次在github -> setting -> 自定义域名填写:用该域名定向到你的博客。

##### 自定义页面
参考[next配置](http://theme-next.iissnan.com/getting-started.html)一一配置即可，下面叙述官网没有的配置。
