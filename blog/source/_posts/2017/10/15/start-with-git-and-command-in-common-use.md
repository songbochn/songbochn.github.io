---
title: 「git学习计划」(一)常用命令
date: 2017-10-15 20:57:44
toc: true
tags:
     - git
     - bash
categories:
    - 计划
---

### git简介

git是现在常用的版本管理软件。在编写代码的时候，经常性的会修改代码，如果不使用软件进行管理，不同的版本就会比较混乱，而且不能自由的在各个版本之间进行切换。git还有个cool的地方，就是可以建立远程仓库。如果你有自己的服务器的话，可以使用服务器作为你的远程仓库，随时将你的工作推送到服务器上。没有服务器也不要紧，可以使用github。来随时随地存储你的项目。

### git安装配置
#### 安装
在centos下，用sudo命令或者root帐号使用以下命令就可以安装git
```bash
yum install git
```
<!--more-->

#### 配置
使用下列命令配置git的用户名和邮箱
```bash
git config --global user.name "名字"
git config --global user.email 邮箱名字
```
--global的含义是全局使用，表示该机器上的git仓库都使用该配置

在输入git config后双击tab键，可以很多配置。自定义你的git

### 常用命令

#### git clone

该命令就是将一个版本库拷贝下来。如下命令
```bash
git clone (-b 分支名) (--depth=1) ssh://用户名@ip或者域名:端口号/地址xx.git
```
括号内的可以不填,加上-b 分支名，表示克隆哪个分支，不填表示默认master分支。

---depth=1表示只克隆最近一次commit。 如果以后想用其他的可以使用git fetch --unshallow或者git pull --unshallow

#### git add

git add 用于将文件添加到暂存区，可以选择性的添加需要的文件，git add . 表示添加当天前目录下的所有文件.

#### git commit

git commit 用于将暂存区的内容提交到一个分支上，使用 -m "信息" 添加这一次提交的描述性信息，做了哪些修改等等。非常有用，建议认真书写。

#### git log
git log用于查看提交历史，我一般常和reset一起使用，reset到某一次commit上
#### git relog
用来记录你的命令提交历史。
#### git show
每次commit 系统都会给一串编码表示唯一性,git show 后面可以加该编码查看修改内容。
#### git push
git push origin 本地分支名 :远程分支名 如果本地分支名是从远程分支clone下来的，那么会自动有追踪关系，只写一次分支名就好了。
#### git pull
pull 相当于fetch+merge 完整命令为git pull orgin 远程分支名:本地分支名，便是取远程分支名的更新与本地分支名合并。
#### git fetch
表示取回远程分支的更新，当远程分支有更新的时候，fetch一下，就可以通过origin/远程分支名来访问最新变化了，对本地分支没有影响。
#### git diff
git diff用于比较差异，接两个commit的编码表示比较两个提交的差异，接两个分支，表示比较两个分支的差异。
在git add之前用git diff,在add之后用 git diff --cached
####  git merge
一般在提PR之前需要进行merge，防止提交合并的时候产生冲突，如果merge错了，可以使用git reset回退版本。
#### git reset
git reset --hard 后接commit的提交编码就可以了，编码不用写全，写6位就可以了。
也可以使用git reset --hard HEAD  HEAD是当前版本，HEAD^ 表示往前在退一个版本，HEAD^^ 再往前推一个版本，往前再推的话可能会数不过来，就可以写成HEAD～100表示往前推100个版本。
#### git branch
git branch用来查看分支，-r 即意为remote 查看远程法分支, -a 意为all 查看所有分支，git branch -D 接分支名，表示删除该本地分支。
#### git checkout
git checkout 后接分支名 切换到该分支上，记得切换前进行提交 -b 表示新建分支并切换到该分支
