---
title: 为什么写博客，以及如何搭建一个独立域名的博客？
date: 2017-09-18 23:37:58
tags: 
    - jekyll 
    - hexo 
    - nginx
categories: 
    - 思考
---

在经过自己的选择困难症之后，目前使用hexo进行博客的搭建。骨子里还是想用jekyll的，觉得那样比较cool一点。

# 为什么要写博客

* 在平常的看书中，有一些自己的思考，平常都是记在手机的便签上，这样会比较杂乱而且不具有系统性的思考，于是便动起了写博客的想法。

* 平常在学习中，不可避免的要去谷歌上找资料，自己习惯性的会收藏下来，方便自己下次查找，这样随着时间的累计，书签里边也会冗余，可以考虑将知识整理下来，方便于自己的查找。

* 最终有一点，我觉得写博客是一件非常cool的一件事，也是在训练自己思维方式的一件事。有了输入才会有输出，英语只有说的清楚，才证明你听懂了。能将事情系统的写下来，就可以证明你理解了。写博客也是一个经营自己的过程，希望能树立自己的品牌也是自己的一个想法。

* 作为记录一个自己时间旅行的地方，不知道大家是否会有过去一年，过去一个月不知道自己都做了些什么，这个时候你需要一个博客，来记录你在这一段时间的旅行，在途中你所做的事情，你的收获。走走，停停，回头看，这样有助于你的进步，我想这也是作为一个工程师所应该有的品质。

<!--more-->

## 这里有一些前辈也说的很好，大家可以参考下。

* [为什么你应该（从现在开始就）写博客 刘未鹏][liuweipeng]

* [为什么要自建博客？ 知乎][blogzhihu]

* [为什么我要写独立域名的博客][whyblog]

* [你应该写博客 —— Steve Yegge][blogsteve]

经过自己的思考之后，就开始进行博客的搭建工作了，下边就是我搭建博客的流程。

# 博客搭建流程

## 域名购买
     
刚开始用过博客园的博客，发现域名带个后缀，非常的不爽。既然现在房价已经这么贵了，房子已经很难买的起了， 那么就在网上给自己创建一个个人空间吧，我是选择[namecheap][namecheap]上购买域名的，选择它的原因是因为它的价格比较稳定，在续费的时候基本是一样的。这是是采用paypal付款的，在购买的时候，要注册一个这个账号，绑定银行卡就可以用了。购买完成后我们就拥有了一个属于自己的域名，以后朋友就可以通过这个域名访问你的博客了。申请一个顶级域名的价格大概在60人民币左右。

## VPS购买

在有了域名之后，我们还需要一个服务器，自己用一台不关机的连接网络的电脑也可以实现。但是这样容易被封端口。成本较高而且比较麻烦。如果只是搭建博客的话，可以用github提供的空间。由于我还需要搭建ss科学上网。以后也打算做一个自己的个人网站，因此买了一个VPS，这里选择的是[搬瓦工][banwagong],因为搬瓦工的性价比比较高，最便宜的一年120左右。还支持支付宝付款。现在的系统都是linux系统，所以要熟悉一些常用的linux命令。

## dns解析

买了VPS后，你会看到一个IP地址，这个就是你服务器的地址，那么别人通过域名访问你的博客的时候，是如何找到你的服务器IP呢？这里就是DNS解析。DNS服务器也可以自己搭建。不过为了方便，我们可以采用国内的DNSpod来作为我们的dns服务上，在namecheap里的nameserver设置为dnspod的地址，可以参照这个[dnspod设置][dns]里对域名进行解析。

## nginx安装

dns解析一般需要72小时内生效，生效之后就可以ping你的域名了，按照上一步配置应该就可以ping通了。然后我们还需要安装nginx作为我们的web服务器，对80或者8080端口进行监听。换句话说，就是当用户访问服务器的时候，nginx负责按照你设置的路径寻找资源并显示给用户。centos7上安装命令如下:

创建一个repository文件 /etc/yum.repos.d/nginx.repo

```bash
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
```

然后执行以下以下命令就完成安装和启动nginx了。

```bash
yum install nginx
启动nginx
service nginx start
```

## hexo和jekyll的选择

这一步我们要解决的问题是如何将我们我们所书写的博文解析成静态文件放置在VPS上，当用户访问的时候，让nginx可以找到。现在有很多的框架，可以参照[这里][kuangjia]，现在所使用的基本是hexo和jekyll。

* hexo是在本地生成好文件推送到你的网站上去，而jekyll是你将你的博文，通常是markdown文件推送到服务器，服务器上监测到
有文件改变后，进行生成。

* 两者所采用的实现语言不同，在文章数不断增多的时候，hexo的生成速率比jekyll快。

* hexo是将主题与博文分开的，而jekyll通常你在下载别人的主题的时候也会将别人的博文一同下载下来。

本来我是在采用jekyll的，看到hexo这个主题可以继承搜索功能，处于先把博客搞出来的目的，就使用了hexo，以后应该会改回jekyll，哈哈。关于hexo，推荐看hexo的[官网][hexo],出自一位台湾大学生之手，这是他写的一篇博文，叫做[hexo飒爽登场][hexoshow],具体的使用，官网都有我就不赘述了，自己慢慢捣腾也是件很幸福的事情。再往后你们就可以踏上博客的旅途啦，希望大家都能有满满的灵感，写出非常好的博文。

[liuweipeng]:http://mindhacks.cn/2009/02/15/why-you-should-start-blogging-now/
[blogzhihu]:https://www.zhihu.com/question/19916345
[whyblog]:https://www.bennythink.com/reason-of-this-blog.html
[blogsteve]:https://www.logcg.com/archives/836.html
[namecheap]:https://www.namecheap.com/
[banwagong]:https://bandwagonhost.com/
[dns]: http://jingyan.baidu.com/article/76a7e409a7c837fc3b6e15e7.html
[kuangjia]:http://topspeedsnail.com/static-website-generators_or_tools/
[hexo]:https://hexo.io/zh-cn/index.html
[hexoshow]:https://zespia.tw/blog/2012/10/11/hexo-debut/
