---
author: admin
comments: true
date: 2011-09-01 12:00:48+00:00
layout: post
slug: code_python_with_vim
title: 用vim写python代码
wordpress_id: 71
categories:
- python
tags:
- vim
---
在自己的home目录下编辑`.vimrc`文件,如果没有新建一个.

下面的配置能让自己过得舒服一点

```shell
" 不要使用vi的键盘模式，而是vim自己的
set nocompatible

" 设定解码
if has("multi_byte")
    " When 'fileencodings' starts with 'ucs-bom', don't do this manually
    "set bomb
    set fileencodings=ucs-bom,utf-8,chinese,taiwan,japan,korea,latin1
    " CJK environment detection and corresponding setting
    if v:lang =~ "^zh_CN"
        " Simplified Chinese, on Unix euc-cn, on MS-Windows cp936
        set encoding=utf-8
        set termencoding=utf-8
        if &fileencoding == ''
            set fileencoding=utf-8
        endif
    elseif v:lang =~ "^zh_TW"
        " Traditional Chinese, on Unix euc-tw, on MS-Windows cp950
        set encoding=euc-tw
        set termencoding=euc-tw
        if &fileencoding == ''
            set fileencoding=euc-tw
        endif
    elseif v:lang =~ "^ja_JP"
        " Japanese, on Unix euc-jp, on MS-Windows cp932
        set encoding=euc-jp
        set termencoding=euc-jp
        if &fileencoding == ''
            set fileencoding=euc-jp
        endif
    elseif v:lang =~ "^ko"
        " Korean on Unix euc-kr, on MS-Windows cp949
        set encoding=euc-kr
        set termencoding=euc-kr
        if &fileencoding == ''
            set fileencoding=ecu-kr
        endif
    endif
    " Detect UTF-8 locale, and override CJK setting if needed
    if v:lang =~ "utf8$" || v:lang =~ "UTF-8$"
        set encoding=utf-8
    endif
else
    echoerr 'Sorry, this version of (g)Vim was not compiled with "multi_byte"'
endif

" 自动格式化设置
filetype indent on
set autoindent
set smartindent

" 显示未完成命令
set showcmd
" 侦测文件类型
filetype on

" 载入文件类型插件
filetype plugin on

" 为特定文件类型载入相关缩进文件
filetype indent on

" 语法高亮
syntax on

" 显示行号
set number

" tab宽度
set tabstop=4
set cindent shiftwidth=4
set autoindent shiftwidth=4

" 保存文件格式
set fileformats=unix,dos

" 文件被其他程序修改时自动载入
set autoread

" 命令行补全
set wildmenu

" 打开文件时，总是跳到退出之前的光标处
autocmd BufReadPost *
 if line("'"") > 0 && line("'"") <= line("$") |
   exe "normal! g`"" |
 endif

filetype plugin on      "允许使用ftplugin目录下的文件类型特定脚本
filetype indent on      "允许使用indent目录下的文件类型缩进

"                            PYTHON 相关的设置                                 "
"Python 文件的一般设置，比如不要 tab 等
"设置自动缩进为4,插入模式里: 插入 <Tab> 时使用合适数量的空格。
"要插入实际的制表，可用 CTRL-V<Tab>
autocmd FileType python setlocal expandtab | setlocal shiftwidth=4 |
    setlocal softtabstop=4 | setlocal textwidth=76 |
    setlocal tabstop=4

"搜索逐字符高亮
set hlsearch
set incsearch

"设置代码样式
colorscheme desert

"设置tags查找位置
set tags=tags;
set autochdir
```

官方地址[http://www.vim.org/scripts/script.php?script_id=850](http://www.vim.org/scripts/script.php?script_id=850),
下载zip包,在home目录下查找`.vim`文件夹,如果没有创建这个目录

官网有安装说明"install details",
完成后`.vim`的文件结构如下:

```shell
.vim
└── after
    └── ftplugin
        ├── pydiction
        │   └── complete-dict
        └── python_pydiction.vim
```

然后配置`.vimrc`,添加语句
`let g:pydiction_location = '~/.vim/after/ftplugin/pydiction/complete-dict'`

后面的值是complete-dict文件的路径,
用vim编辑一个py文件,`import os.<TAB>`,这时候应该出现提示,证明成功了.

`ctrl+n ctrl+p`选择列表里的提示项

其实7.2版本的vim自身已经提供了比较强悍的补全功能, vim的OMNI补全(也叫"全能补全")

`os.<CTRL+x , CTRL+o>`,如果开启了vim的python模块,现在应该有一个分割窗口显示函数的参数,以及__doc__信息

如果需要动态输入刷新提示内容,在配置文件中加入
`set completeopt=longest,menu`

ctags提示

首先确定安装了ctags软件,运行tags生成脚本

```shell
ctags -R *.py`
```

这时会生成tags文件

安装taglist插件,网址[http://www.vim.org/scripts/script.php?script_id=273](http://www.vim.org/scripts/script.php?script_id=273)

安装完成后,编辑py文件,执行vim命令`:Tlist`,
会出现taglist窗口,如果需要tags文件中的关键词补全,`CTRL+n`,如果需要跟踪关键词文件`CTRL+]`,跳回来`CTRL+t`

代码模板,主页地址 [http://www.vim.org/scripts/script.php?script_id=2540](http://www.vim.org/scripts/script.php?script_id=2540)

安装后,这个插件默认的快捷键是`<TAB>`

但是pydiction_location默认的快捷键也是`<TAB>`,这里修改 pydiction_location的快捷键

找到`.vim/after/ftplugin/python_pydiction.vim`文件,
修改

```shell
" Make the Tab key do python code completion:
inoremap <silent> <buffer> <TAB>
```

为

```shell
" Make the Tab key do python code completion:
inoremap <silent> <buffer> <C-P>
```

这样就把pydiction_location的快捷键修改为`CTRL+p`了,
然后编辑py文件,输入 `cl<TAB>`,就会出现class的定义模板了,
这些模板定义在`.vim/syntax`文件夹下,可自行修改.

py语法检查插件 [http://www.vim.org/scripts/script.php?script_id=2441](http://www.vim.org/scripts/script.php?script_id=2441)

安装以后会用红色,提示py代码的错误
