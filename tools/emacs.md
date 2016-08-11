# 0x00 catalog

这个post记录使用`emacs`时候一些工具.
最前面吐血安利spacemacs,简单省事,感觉很不错.主要功能通过layer(类似配置文件)划分,帮助你节省配置时间.

## 0x10 keybind

之前使用vs时候喜欢F5去编译一运行一个程序，emacs的默认配置里面不提供这样的功能。
可以通过c mode hook bind F5 到`compile`命令来调用`makefile`:

```lisp
(add-hook 'c-mode-hook'
          (lambda ()
            (local-set-key (quote [f5]) (quote compile))))
```

## 0x20 auto complete

不同语言的补全记录.

### 0x21 C/C++

layer里面对C/C++的补全提供了ycmd-client,这个插件服务器端还是需要自己编译安装,且ycmd每一个project都有一个叫`.ycm_extra_conf.py`的配置文件,其可以通过YCM-Generator来生成.

### 0x22 python

>It aims to provide an easy to install, fully-featured environment for Python development.

python补全发现非常棒的插件elpy!(统筹了很多功能,被动技能最棒)

* 被动技能-PEP8风格检查
* 语法检查
* 自动补全
* 项目管理
...

### 0x23 go

在`.spacemacs`里面开启`go-mode`和相关`require`.然后安装一些必要的可执行问题,且要配置好`go`相关的变量.

```
(require 'go-autocomplete)
(require 'auto-complete-config)
```
go的环境变量

```
...
GOPATH="/Users/guowang/workspace/go"
GOROOT="/usr/local/Cellar/go/1.6.2/libexec"
...
```

几个在$GOPATH/bin下面几个会被emacs调用的工具需要通过go预先安装:`cover     gocode    goimports gorename  oracle`.
默认安装遇到一个小坑就是不能补全第三方库,通过gocode的的设置可以修改

```
➜  ~ gocode set
propose-builtins true
lib-path ""
autobuild false
force-debug-output ""
package-lookup-mode "go"
close-timeout 1800
```

## 0x30 utils

### 0x31 irc

spacemacs自带`erc`的layer,修改一行代码就能自动跑起来irc客户端,很是方便.而且可以emacs变成服务一直在后台这样irc就不用下线了(记得AFK).

### reference

[1] [Emacs FAQ](https://www.gnu.org/software/emacs/manual/html_node/efaq/Binding-keys-to-commands.html)
[2] [Elpy](http://elpy.readthedocs.io/en/latest/concepts.html)
