---
title: 如何在mac上搭建tmux并进行系统化配置
date: 2018-07-06 19:16:27
tag: 
	- 终端
	- bash
---
平常使用终端的时候，需要开各种各样的窗口。不但使得桌面杂乱无章，而且一点也不酷。我就想肯定有更帅气的解决方案，在网上搜了下果然发现个神器tmux，想入坑命令行的同学看过来。网上各种博客上的资料比较杂乱，这里做一个系统化的汇总。

关于学习视频可以参考[tmux教程][tmux]

本文标题是在mac上，实际上只有安装方式不同，配置都是一样。

那么，let do it!

####安装
```bash
brew install tmux
```
<!--more-->
就这么一句简单的命令就可以搞定了。

下面简单介绍下tmux，它最基础的概念是session(会话)，可以把这个理解为电脑上不同的程序，在一个会话下可以有多个window(窗口)，可以当做一个程序不同的功能页面来理解。一个窗口下可以有多个panel(面板)，一个window就是一整个屏幕了，那么你把这个屏幕分成几个区域，就会有几个面板，在不同的面板可以做不同的事情。

安装完后在命令行输入tmux就可以使用了，tmux的快捷键都需要先按下前缀Ctrl-b然后释放，再按对应的按键做对应的动作。

我试着操作了一下，发现这也太傻了吧。肯定有优化的地方,网上都说换成Ctrl-a,可是我觉得为什么不能用一个方便的键来代替呢，于是就有了下面的配置。

首先打开tmux的配置文件

```bash
vim ~/.tmux.conf
```
如果是刚安装tmux的话你打开应该是空的，这时候就填入自己喜欢的配置了，大改特改。
首先就是前缀键，改掉! #后边

```bash
unbind C-b # 解绑原有的前缀键盘
set -g prefix ` #
bind ` send-prefix
```

现在按下`加上对应的快捷键就可以进行操作

其他配置可以参考以下配置

```bash
# Use Alt-arrow keys to switch panes
bind -n M-Left select-pane -L
bind -n M-Right select-pane -R
bind -n M-Up select-pane -U
bind -n M-Down select-pane -D

# Shift arrow to switch windows
bind -n S-Left previous-window
bind -n S-Right next-window

# Set easier window split keys
bind-key v split-window -h
bind-key h split-window -v

# Easy config reload
bind-key r source-file ~/.tmux.conf \; display-message "tmux.conf reloaded"
```

就可以使用alt加方向键来选择panel，shift加左右键来选择window了。创建window也只需要\`-h或者\`-v来触发就可以了。
####附录
+ [快捷键列表][shortkey]
+ [tmux系列][tmuxteach]

####参考资料
https://zanshin.net/2014/12/27/changing-my-tmux-command-prefix-to-tic/

	
[tmux]:https://www.youtube.com/watch?v=nD6g-rM5Bh0
[shortkey]:https://tmuxcheatsheet.com/
[tmuxteach]:https://minimul.com/teaches/tmux