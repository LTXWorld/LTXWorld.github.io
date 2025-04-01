+++
date = '2025-04-01T08:36:54+08:00'
draft = true
title = 'Tmux终端窗口器'
+++

## 引子

开篇提问，大家平时用终端的次数多吗？

![](/img/xm/hdw.png)

如果大多数时候你在虚拟机上工作，那不用多说，如果是macos，估计iTerm你已经耳熟能详，如果是windows，也许是Powershell?（本人平时windows只用来打游戏了）

不管怎么样，在我们使用这些终端的时候，通常都会面对一个大大的黑框，然后就在这个黑框里不断地操作，那么此时如果我想要保留当前框中的内容的同时又想开另外一个框来进行其他操作呢？我想，这一定是所有使用终端的人都会遇到的问题。

系统中的默认终端编辑器给我们了这样一种做法：New Tab，与浏览器窗口的打开一样，一个接着一个，但这还不够好，如果我想要同时观察两个窗口中的内容呢？总不能一会儿点这个，一会儿又点回另一个吧，那这样的操作肯定是有点累赘的。

所以就有了tmux这个终端窗口器，它可以做到在一个窗口内，分割出多个子窗口，这个子窗口其实叫做窗格，接下来我们会简单的说一下其中的概念，不过我想当你亲自上手使用一段时间后，这些概念反倒没有那么重要了。

![](/img/riscv/tmux3.png)

我对于tmux的了解来源于[Missing课程](https://missing-semester-cn.github.io/)，其中也包含了Vim的操作等非常重要的课程。

## 简介

Tmux中有几个概念需要了解，从大往小来说分别是：会话->窗口->窗格
下面我们逆序来分别介绍一下他们。

### 窗格

如上图所示，窗格就是我在一个窗口内分出的四个块，甚至可以做到不同尺寸（稍后见配置文件），这每一个窗格我们其实都可以把他当作一个新的终端，这样就可以完美解决在引子中我们提到的问题，可以同时观察终端中运行的不同命令的结果。

我们可以垂直划分也可以水平划分窗口来创建不同的窗格。（操作见下方快捷键）

### 窗口

窗口就是最开始普通的终端编辑器中的大窗口的概念，在tmux中我们不仅可以在一个窗口内创建多个窗格，还可以在一个会话内创建多个窗口，就像下图中红色框出的部分，在这一个会话中我创建了5个不同的窗口并且对其进行不同的命名以区分每个窗口的作用，是不是比不停的去New Tab要整齐地多？

![](/img/riscv/tmux4.png)

这些窗口已经自动排好了序，那如何在这些窗口中切换呢？虽然tmux给了相应的快捷键，但是我更改了配置文件，现在可以做到直接点击切换窗口。

### 会话

我们上面说过，一个会话包含多个窗口，而我们每次进行操作本质上是进入了一个会话中，我们一般以一个会话包含一系列类似的工作窗口。

如何创建和退出会话呢，这里就要用到一些命令了(当然快捷键也可以)

```bash
# 新建
tmux new -s <会话名>
# 退出会话，但是仍在后台执行没有销毁
tmux detach
# 进入会话
tmux attach -t <会话名>
# 列出会话
tmux ls
# 杀死
tmux kill-session -t <会话名> or <会话id>
# 切换会话
tmux switch -t <会话名> or <会话id>
# 重命名
tmux rename-session -t <会话名> <new-name>
```

## 配置文件

tmux本身是一个快捷键很多的工具，而其快捷键又极其依赖操作前缀，默认是`Ctrl/Command+b`,而低头看看自己的键盘，这两个键离得是不是有点稍远了点，不是那么好按啊。以及其新建窗格的按键是什么`%`,切换又是`;`这种有些很难记且很难按的设置，所以我们需要个性化一下，来改变`~/.tmux.conf`这个文件的内容。

下面我列出的配置是专门对于我macos的配置，其实与windows或者linux大查不查，有些地方不一样而已。

- 将前缀修改为`Ctrl/Command+a`,下面以Pre代替

**窗口管理**

- 启用`base-index 1`让窗口编号从1开始
- `Pre+c` 新建窗口，`Pre+t`切换下一个窗口；`Pre+space/Backspace`下/上一个窗口
- `Pre+v/s`垂直/水平分割当前窗口产生窗格
- `Pre+,`对窗口进行重命名操作

**窗格管理**

- 启用了vim模式，可以使用`Pre + h/j/k/l`在窗格之间导航
- `Pre+a`切换到上一个窗格
- `setw -g mouse on`启用了鼠标模式，这样我们可以实现点选窗口，拖拽窗格大小，还能够选择一段代码（其状态会变为黄色）自动复制到剪贴板上，方便复制（注意这样点选后会自动来到最新的终端位置）
- 并且开启了状态栏，可以在底部显示时间信息。

其他的操作大家可以自行查阅资料。

```bash
# vim style tmux config

# use C-a, since it's on the home row and easier to hit than C-b
set-option -g prefix C-a
unbind-key C-a
bind-key C-a send-prefix
set -g base-index 1

# Easy config reload
bind-key R source-file ~/.tmux.conf \; display-message "tmux.conf reloaded."

# vi is good
setw -g mode-keys vi

# mouse behavior
setw -g mouse on

set-option -g default-terminal screen-256color

bind-key : command-prompt
bind-key r refresh-client
bind-key L clear-history

bind-key space next-window
bind-key bspace previous-window
bind-key enter next-layout

# use vim-like keys for splits and windows
bind-key v split-window -h
bind-key s split-window -v
bind-key h select-pane -L
bind-key j select-pane -D
bind-key k select-pane -U
bind-key l select-pane -R

# smart pane switching with awareness of vim splits
bind -n C-h run "(tmux display-message -p '#{pane_current_command}' | grep -iqE '(^|\/)vim$' && tmux send-keys C-h) || tmux select-pane -L"
bind -n C-j run "(tmux display-message -p '#{pane_current_command}' | grep -iqE '(^|\/)vim$' && tmux send-keys C-j) || tmux select-pane -D"
bind -n C-k run "(tmux display-message -p '#{pane_current_command}' | grep -iqE '(^|\/)vim$' && tmux send-keys C-k) || tmux select-pane -U"
bind -n C-l run "(tmux display-message -p '#{pane_current_command}' | grep -iqE '(^|\/)vim$' && tmux send-keys C-l) || tmux select-pane -R"
bind -n 'C-\' run "(tmux display-message -p '#{pane_current_command}' | grep -iqE '(^|\/)vim$' && tmux send-keys 'C-\\') || tmux select-pane -l"
bind C-l send-keys 'C-l'

bind-key C-o rotate-window

bind-key + select-layout main-horizontal
bind-key = select-layout main-vertical

set-window-option -g other-pane-height 25
set-window-option -g other-pane-width 80
set-window-option -g display-panes-time 1500
set-window-option -g window-status-current-style fg=magenta

bind-key a last-pane
bind-key q display-panes
bind-key c new-window
bind-key t next-window
bind-key T previous-window

bind-key [ copy-mode
bind-key ] paste-buffer

# Setup 'v' to begin selection as in Vim
bind-key -T copy-mode-vi v send -X begin-selection
bind-key -T copy-mode-vi y send -X copy-pipe-and-cancel "xclip -sel clip"

# Update default binding of `Enter` to also use copy-pipe
unbind -T copy-mode-vi Enter
bind-key -T copy-mode-vi Enter send -X copy-pipe-and-cancel "xclip -sel clip"

# Status Bar
set-option -g status-interval 1
set-option -g status-style bg=black
set-option -g status-style fg=white
set -g status-left '#[fg=green]#H #[default]'
set -g status-right '%a%l:%M:%S %p#[default] #[fg=blue]%Y-%m-%d'

set-option -g pane-active-border-style fg=yellow
set-option -g pane-border-style fg=cyan

# Set window notifications
setw -g monitor-activity on
set -g visual-activity on

# Enable native Mac OS X copy/paste
set-option -g default-command "/bin/bash -c 'which reattach-to-user-namespace >/dev/null && exec reattach-to-user-namespace $SHELL -l || exec $SHELL -l'"

# Allow the arrow key to be used immediately after changing windows
set-option -g repeat-time 0

```

最终激活配置可以直接`Pre+R`这是我们设置好的激活快捷键；或者你直接重启tmux服务

```bash
tmux kill-server，
tmux
```

## 简单流程

总结一下使用的流程，来到我们的终端后，执行如下操作

```bash
tmux new -s t1
# 新建一些窗口
Pre+c
# 对当前窗口进行垂直或水平划分
Pre+v/c
# 进行终端操作即可
# 退出tmux会话
tmux detach
# 再次进入
tmux ls
tmux a -t <会话名>
```
