---
title: Vim配置之通用配置
author: 张帆
tags:
  - vim
date: 2018-08-05 13:28:23
---

Vim一个最突出的特点就是它强大的可定制性。但过度的自由也为初学者带来了一大困扰，配置太多太杂，网上的各种配置文章良莠不齐，从茫茫的配置项中找出适合自己的配置方案，完全不知道如何下手。
由于不同人有不同的习惯，因此并没有一套通用的适合所有人的配置方案，这里我仅以我常用的通用配置文件为例进行一个讲解。
Vim的通用配置指的是不依赖插件的一些配置，这些配置一般可被IDE等的Vim插件所支持，因此我称之为通用配置。Vim的通用配置主要分为选项，快捷键和自动命令三部分。

<!--more-->

## 选项

选项是对Vim的特定行为的效果进行控制的配置，通常使用`set`指令进行配置。
``` vim
" ++++++++++++++++++++++++++++++++++++++++
" +             基础选项配置             +
" ++++++++++++++++++++++++++++++++++++++++

" 非兼容模式
set nocompatible

" 显示行号
set number
set relativenumber

" 搜索设置
set ignorecase
set incsearch
set hlsearch

" 开启鼠标
set mouse=a

" 切换缓冲区文件可以不保存当前文件
set hidden

" 默认开启语法高亮
syntax on

" tabs
set expandtab
set softtabstop=4
set shiftwidth=4
set tabstop=4

" 缩进设置，设置基于文件的类型的缩进
set autoindent
set cino=(0,W4
filetype plugin indent on

" 改变超过 80 个字符之后的区域，这样就可以起到提示作用
set colorcolumn=81
set fo+=mB " 支持中文
set wrap

" 自动载入外部修改
set autoread

" 关闭延时
set ttimeoutlen=0

" 编码设置
set encoding=utf-8
set fileencodings=utf-8,ucs-bom,GB2312,big5

" 高亮当前行
set cursorline

" 智能补全命令行
set wildmenu
set wildmode=full

" 不显示状态，airline已有
set noshowmode

" 代码折叠
set foldenable              " 开始折叠
set foldmethod=indent       " 设置缩进折叠
set foldcolumn=0            " 设置折叠区域的宽度
setlocal foldlevel=1        " 最大只折叠一层
set foldlevelstart=99       " 打开文件默认不折叠代码

" 删除设置
set backspace=eol,start,indent

" 设置隐藏字符, 通过 set list 显示
set listchars=tab:▸\ ,eol:¬

" 字体
set guifont=FuraCode\ Nerd\ Font\ Mono\ 10

if !isdirectory($HOME."/.undo_history")
    call mkdir($HOME."/.undo_history", "", 0700)
endif
" undo 历史保存路径
set undodir=~/.undo_history/
" 开启保存 undo 历史功能
set undofile
```

## 快捷键

无论在哪个编辑器或IDE中，熟练掌握快捷键都是提高效率一个必不可少的途径。在Vim中，我么可以将自己常用的命令映射为快捷键，提高效率。此类配置项一般以包括`map`的命令开通，如`noremap`，`vnoremap`，`cnoremap`等。

``` vim
" 修改leader键
let mapleader = ';'
let maplocalleader = ','

" 绑定 jk <Esc>，这样就不用按角落里面的 <Esc>
inoremap jk <Esc>

"绑定大写的 HL 为行首和行尾的快捷键
noremap H ^
noremap L $

" 交换 ' 和 ` 的功能
nnoremap ' `
nnoremap ` '
noremap + ;
noremap - ,


" 使用 %% 扩展当前文件的路径
cnoremap <expr> %% getcmdtype() == ':' ? expand('%:h').'/' : '%%'

" 使得 & 命令能够重复上一次的命令，包括 flag
nnoremap & :&&<CR>
xnoremap & :&&<CR>

" 命令行模式 Ctrl-j 下一条命令，Ctrl-k 上一条命令
cnoremap <C-j> <Down>
cnoremap <C-k> <Up>
cnoremap <C-h> <Left>
cnoremap <C-l> <Right>
cnoremap <C-a> <Home>
cnoremap <C-e> <End>

" 系统剪贴板复制与粘贴
nnoremap <C-p> "+gp
vnoremap <C-p> "+gp
vnoremap <C-y> "+y

" 复制
nnoremap <Leader>y yiw

" 打开/关闭quickfix
nnoremap <leader>q :cclose<cr>

" j/k在没有计数的时候按虚拟行移动，有计数的时候按实际行移动
nnoremap <expr> j v:count ? 'j' : 'gj'
nnoremap <expr> k v:count ? 'k' : 'gk'

nnoremap <silent> <Space> @=(foldlevel('.')?'za':"\<Space>")<CR>
nnoremap <silent> <Backspace> :nohl<CR>

" 使用超级用户权限编辑这个文件
cmap w!! w !sudo tee >/dev/null %

" 调整宽度
cmap v= vertical resize +5<cr>
cmap v- vertical resize -5<cr>
cmap s= resize +5<cr>
cmap s- resize -5<cr>
```

## 自动命令

自动命令是Vim中特有的，指的是在某些事件发生后自动执行的命令，一般以`autocmd`开始。

``` vim
" 将光标设为上次退出时的位置
autocmd BufReadPost *
    \ if line("'\"") > 1 && line("'\"") <= line("$") |
    \   exe "normal! g`\"" |
    \ endif

" K to lookup current word in cppman
command! -nargs=+ Cppman silent! call system("tmux split-window cppman " . expand(<q-args>))
autocmd FileType cpp nnoremap <silent><buffer> K <Esc>:Cppman <cword><CR>

autocmd BufNewFile *.cpp,*.c,*.h,*.hpp 0r ~/.license
" 使qss文件可以被css文件插件支持
autocmd BufNewFile,BufFilePre,BufRead *.qss set filetype=css
" vim 类型文件设置折叠方式为 marker
autocmd FileType vim set foldmethod=marker
```

## 小结

Vim的选项很多，我们需要根据自己的习惯和使用场景做出调整。快捷键和自动命令有着极高的自由度，我们在日常使用过程中需要随时留意哪些操作是可以通过配置快捷键和自动命令进行改善的，这样我们使用Vim才会越来越顺手，玩出自己的精彩
