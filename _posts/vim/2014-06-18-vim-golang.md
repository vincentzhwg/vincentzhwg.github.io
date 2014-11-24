---
title: 配置vim搭建golang开发环境
date: 2014-06-18 10:50:00 +0800
tags:
- vim
- golang
- go
---

* toc
{:toc}

### 安装vundle

首先安装vundle，方便进行插件管理

    mdkir -p ~/.vim/bundle/
    git clone https://github.com/gmarik/vundle.git ~/.vim/bundle/vundle






然后在 ~/.vimrc 文件的前面添加如下内容

    set nocompatible               " be iMproved
    filetype off                   " required!
    
    set rtp+=~/.vim/bundle/vundle/
    call vundle#rc()
    
    " let Vundle manage Vundle
    " " required! 
    Bundle 'gmarik/vundle'
    filetype plugin indent on
    syntax on
    
### go环境搭建

在开始下面的步骤前先搭建好系统上相应的go环境，设置好相应的GOROOT及GOPATH，并把GOROOT及GOPATH的bin路径加入到PATH中。
另外，在下面的一些安装步骤中，有些包或库要从google的代码库中下载，所以网络连通问题，各位自行解决哈，不然就会导致下载不了了。

### 安装google的官方vim库

在.vimrc里添加：

    Bundle 'cespare/vim-golang'

然后，启动vim,运行命令 :BundleInstall 安装，到其显示 Done 即可。

### goimports

此插件用于解决麻烦的golang的import问题，其会自动化加入或去除未使用的包，但只限于标准库的包，其余包还是得手动管理。
运行命令下载安装goimports的库

    go get -v code.google.com/p/go.tools/cmd/goimports

完成后，在 .vimrc 中添加以下语句，进行关联

    if exists(#b:did_ftplugin_go_fmt#)
        finish
    endif
    
    if !exists(#g:go_fmt_commands#)
        let g:go_fmt_commands = 1 
    endif
    
    if !exists(#g:gofmt_command#)
        let g:gofmt_command = #goimports#
    endif
    
    if g:go_fmt_commands
        command! -buffer Fmt call s:GoFormat()
    endif
    
    function! s:GoFormat()
        let view = winsaveview()
        silent execute #%!# . g:gofmt_command
        if v:shell_error
            let errors = []
            for line in getline(1, line('$'))
                let tokens = matchlist(line, '^\(.\{-}\):\(\d\+\):\(\d\+\)\s*\(.*\)')
                if !empty(tokens)
                    call add(errors, {#filename#: @%, 
                                     \#lnum#:     tokens[2],
                                     \#col#:      tokens[3],
                                     \#text#:     tokens[4]})
                endif
            endfor
            if empty(errors)
                % | # Couldn't detect gofmt error format, output errors
            endif
            undo
            if !empty(errors)
                call setloclist(0, errors, 'r')
            endif
            echohl Error | echomsg #Gofmt returned error# | echohl None
        endif
        call winrestview(view)
    endfunction
    let b:did_ftplugin_go_fmt = 1

这样，以后在编写代码时，只需运行 `:Fmt` 命令即可自动化import标准包了，解决go语言的一大烦恼呀。

### go-def

代码查找时的自动跳转，如查看函数或结构体的定义。

    go get -v code.google.com/p/rog-go/exp/cmd/godef

在 .vimrc 里添加

    Bundle 'dgryski/vim-godef'

然后，启动vim,运行命令 :BundleInstall 安装，到其显示 Done 即可。
使用方法：打开一个Go代码文件，把光标移到一个函数上，在命令模式下输入gd。vim会显示这个函数的定义。详细配置参见 [godef](https://github.com/dgryski/vim-godef){:target="_blank"}

### gocode:代码补全提示

此插件可提示一些函数用法，及变量名称的补全，个人感觉后一种用途还更实用些

    go get -u github.com/nsf/gocode

在 ~/.vimrc 中添加如下内容

    Plugin 'nsf/gocode', {'rtp': 'vim/'}

然后，启动 vim,运行 :PluginInstall 命令，待其安装完成。
使用方法： 在需要提示时，<C-x><C-o>即可。

### gotags:标签边栏

此插件可在边栏显示当前文件包含哪些函数定义，结构体定义等等.

    go get -u github.com/jstemmer/gotags

在 .vimrc 里添加

    Bundle 'majutsushi/tagbar'

然后，启动vim,运行命令 :BundleInstall 安装，到其显示 Done 即可。

在 .vimrc 文件里添加如下内容


    nmap <F8> :TagbarToggle<CR>
    let g:tagbar_type_go = { 
        \ 'ctagstype' : 'go',
        \ 'kinds'     : [
            \ 'p:package',
            \ 'i:imports:1',
            \ 'c:constants',
            \ 'v:variables',
            \ 't:types',
            \ 'n:interfaces',
            \ 'w:fields',
            \ 'e:embedded',
            \ 'm:methods',
            \ 'r:constructor',
            \ 'f:functions'
        \ ],
        \ 'sro' : '.',
        \ 'kind2scope' : {
            \ 't' : 'ctype',
            \ 'n' : 'ntype'
        \ },
        \ 'scope2kind' : {
            \ 'ctype' : 't',
            \ 'ntype' : 'n'
        \ },
        \ 'ctagsbin'  : 'gotags',
        \ 'ctagsargs' : '-sort -silent'
    \ }


使用方法：按 F8 或 输入命令 `:TagbarToggle` ，即可显示出边栏。
