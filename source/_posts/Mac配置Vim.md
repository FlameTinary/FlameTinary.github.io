title: Mac配置Vim
date: 2019-08-28 18:51:29
tags:
- Mac
categories:
- Mac
---

> 相信不少同学在Mac上使用过`vi`或者`vim`命令，这个命令是打开mac自带的vim编辑器的命令，如果只是简单的修改文件，使用vim无疑是很方便的；但是如果使用mac自带的vim来做大量文档编辑的工作，那就可能力不从心了。
> 这个时候，你就需要更复杂的vim编辑器来负担这样的工作了,Mac目前有两个Vim客户端可供我们使用
> 1、**MacVim：**使用Cocoa GUI，这是Mac上更新还很活跃的版本，也是Mac上最多人使用的版本。[下载地址](https://github.com/macvim-dev/macvim)
> 2、使用Carbon GUI的版本，但是这个版本目前基本上不再更新。[下载地址](http://sourceforge.net/projects/macosxvim/files/)

<!--more-->

我主要是使用的**MacVim**，所以就记录一下MacVim的安装以及使用过程。

## 安装

从上面提供的链接下载MacVim，并按照安装mac应用的步骤安装好MavVim。这个时候还没有结束。

想象一下，我们以前在terminal中打开一个文件是这样的：`vi <文件名称>`，这样很coooooooooool；

但是，我们现在使用MacVim，却得鼠标点击打开MacVim，然后再从MacVim打开文件。这样是不是比以前繁琐了很多。那么，能不能像从前一样，使用命令打开文件呢？

答案是可以的。

我们先把MacVim打开,第一次打开MacVim是这个样子：

![第一次打开MacVim](https://s2.ax1x.com/2019/08/28/m7xckQ.jpg)

在MacVim的界面中我们输入`:h mvim`，MacVim会提示我们将`mvim`命令的路径添加到我们的PATH

![提示](https://s2.ax1x.com/2019/08/28/m7xgYj.jpg)

**添加PATH步骤：**

1. `vi ~/.bash_podfile`打开PATH所在路径
2. 将MacVim提供的命令路径写入PATH：`export PATH=/Applications/MacVim.app/Contents/bin:$PATH`
3. 执行`source ~/.bash_podfile`重启bash

这个时候，我们就可以直接在Terminal中使用`mvim`命令来打开文件了。

是不是觉得输入mvim命令还是不如vim命令舒服？那么我们就使用vim来打开MacVim!

使用命令`mvim ~/.zshrc` 或者`mvim ~/.bashrc`，如果你使用的是zsh那么使用前者，如果使用的是bash，那么使用后者。

打开后，在文件末尾新增一段文字`alias vim='mvim'`，意思就是给mvim命令起一个别名。

然后保存退出，如果你是zsh使用`source ~/.zshrc`重新载入，如果是bash使用`source ~/.bashrc`载入。

接下来就是见证奇迹的时刻了，输入命令`vim <你要打开的文档>`，看看是不是使用MacVim打开了!

## 添加插件

> 经过上面的一系列骚操作，现在我们可以使用MacVim正常的进行编辑了，但是，我们还可以对MacVim进行更深层次的扩展，那就是给MacVim添加插件！

### Vundle 命令

```bash
# 安装插件
:BundleInstall
# 更新插件
:BundleUpdate
# 清除不需要的插件
:BundleClean
# 列出当前的插件
:BundleList
# 搜索插件
:BundleSearch

```

## 配置~/.vimrc文件

```bash
" 关闭vim的所有扩展功能
set nocompatible

" 配置插件路径
set rtp+=~/.vim/bundle/vundle/  
call vundle#begin()

" 在这里添加你想安装的Vim插件
Bundle 'gmarik/vundle'
" Python补全强力插件
Bundle 'davidhalter/jedi'
" 添加引号,括号配对补全
Bundle 'jiangmiao/auto-pairs'
" 添加/解除注释
Bundle 'scrooloose/nerdcommenter'

call vundle#end()       " required

"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" 一般设定
"""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
" 设定默认解码
set fenc=utf-8
set fencs=utf-8,usc-bom,euc-jp,gb18030,gbk,gb2312,cp936

"去掉讨厌的有关vi一致性模式，避免以前版本的一些bug和局限
set nocompatible

" 显示中文帮助
if version >= 603
    set helplang=cn
    set encoding=utf-8
endif

" 语法高亮
syntax on

" 设置配色
colorscheme solarized  "可能需要提前下载

" 设置字体
set guifont=Monaco:h14

" 设置gvim启动窗口的位置，以及大小
" winpos 300 105
" set lines=30 columns=100

" 开启行号显示
set number

"下面两行在进行编写代码时，在格式对起上很有用；
"第一行，vim使用自动对起，也就是把当前行的对起格式应用到下一行；
"第二行，依据上面的对起格式，智能的选择对起方式，对于类似C语言编
"写上很有用
set autoindent
set smartindent

"查询时非常方便，如要查找book单词，当输入到/b时，会自动找到第一
"个b开头的单词，当输入到/bo时，会自动找到第一个bo开头的单词，依
"次类推，进行查找时，使用此设置会快速找到答案，当你找要匹配的单词
"时，别忘记回车
set incsearch

" 高亮当前行
set cursorline

" 启动的时候不显示那个援助索马里儿童的提示
set shortmess=atI

" 我的状态行显示的内容（包括文件类型和解码）
set statusline=%F%m%r%h%w\[POS=%l,%v][%p%%]\%{strftime(\"%d/%m/%y\ -\ %H:%M\")}

" 总是显示状态行
set laststatus=2

" 制表符为4
set tabstop=4

" 统一缩进为4
set softtabstop=4
set shiftwidth=4

" 在c,c++,python文件中用空格代替制表符
autocmd FileType c,cpp,python set shiftwidth=4 | set expandtab

" 侦测文件类型
filetype on

" 载入文件类型插件
filetype plugin on

" 为特定文件类型载入相关缩进文件
filetype indent on
   " 如果文件类型为.sh文件
    if &filetype == 'sh'
        call setline(1,
        "\#########################################################################")
        call append(line("."), "\# File Name: ".expand("%"))
        call append(line(".")+1, "\# Author: Sheldon")
        call append(line(".")+2, "\# mail: tinarychina@gmail.com")
        call append(line(".")+3, "\# Created Time: ".strftime("%F %R"))
        call append(line(".")+4, "\#########################################################################")
        call append(line(".")+5, "\#!/bin/bash")
        call append(line(".")+6, "")

    else
        call setline(1, "/*************************************************************************")
        call append(line("."), "    > File Name: ".expand("%"))
        call append(line(".")+1, "    > Author: Sheldon")
        call append(line(".")+2, "    > Mail: tinarychina@gmail.com ")
        call append(line(".")+3, "    > Created Time: ".strftime("%F %R"))
        call append(line(".")+4, " ************************************************************************/")
        call append(line(".")+5, "")
    endif
    if &filetype == 'python'
        call append(line(".")+5, "\#!/usr/bin/env python")
        call append(line(".")+6, "\#coding: utf-8")
        call append(line(".")+7, "")
    endif
    if &filetype == 'cpp'
        call append(line(".")+6, "#include<iostream>")
        call append(line(".")+7, "using namespace std;")
        call append(line(".")+8, "")
    endif
    if &filetype == 'c'
        call append(line(".")+6, "#include<stdio.h>")
        call append(line(".")+7, "")
    endif
    " 新建文件后，自动定位到文件末尾
    autocmd BufNewFile * normal G
endfunc
```

## vim 安装主题

在`~/.vim/`路径下新建一个**colors**文件夹，然后下载主题，例如：https://github.com/tomasr/molokai，将下载好的molokai.vim放入colors文件夹中。

使用vim打开`~/.vimrc`，并添加：
```
" 代码颜色主题
syntax on
syntax enable
set t_Co=256
colorscheme molokai
```

然后重新打开vim编辑器，就发现主题已经更换了
