# 环境工具之Ubuntu使用记录

**一、apt命令使用**

apt 命令是一个功能强大的命令行工具，它不仅可以更新软件包列表索引、执行安装新软件包、升级现有软件包，还能够升级整个 Ubuntu 系统\(apt 是 Debian 系操作系统的包管理工具\)。

与更专业的 APT\(Advanced Packaging Tool\) 工具 apt\-get 和 apt\-cache 相比，apt 具有一些更适合交互式场景的选项，它更倾向于成为面向最终用户的工具\(而不仅仅是系统管理员\)。换句话说，apt 比 apt\-get 用起来更简单，用户体验更好。

本文介绍 apt 命令的基本用法，演示环境为 Ubuntu 18.04。

基本语法

语法格式：

apt \[options\] command

配置文件：

早期 apt 默认的配置文件为 /etc/apt/apt.conf，但是当前的 Ubuntu 系统中默认没有这个文件。

如果 /etc/apt/apt.conf 文件存在，apt 仍然会读取它。但现在的设计思路是把配置文件分隔后放置在 /etc/apt/apt.conf.d 目录下，这样更容易管理。

常用子命令：

update

update 命令用于从配置的源下载包信息。update 命令应该总是在安装或升级包之前执行。

upgrade

upgrade 命令用于从配置的源安装当前系统中的所有包的可用升级。如果需要满足依赖关系，就安装新的包，但是不会删除现有的包。如果包的升级需要删除已安装的包，则不执行此包的升级。

full\-upgrade

full\-upgrade 命令执行升级功能，如果需要将系统升级到新的版本，则会删除当前已安装的包。

install，remove，purge

install 命令用来安装一个或多个指定的包。remove 命令用来删除包，但是会保留包的配置文件。purge 命令会在删除包的同时删除其配置文件。

autoremove

autoremove 命令用于删除自动安装的包，这些包是为了满足其他包的依赖关系而自动安装的，随着依赖关系的更改或需要它们的包已被删除，这些包现在不再需要了。

search

search 命令用于在可用包列表中搜索给定的项并显示匹配到的内容。例如，如果您正在寻找具有特定功能的包，这将非常有用。

show

show 命令显示关于给定包的信息，包括它的依赖关系、安装和下载大小、包的来源、包内容的描述等等。比如，在删除一个包或搜索要安装的新包之前查看这些信息是很有帮助的。

list

list 命令可以显示满足特定条件的包列表，默认列出所有的包。可以通过 \-\-installed 选项列出已安装的包，\-\-upgrade 选项列出可以升级的包。

edit\-sources

edit\-sources 命令用来编辑 /etc/apt/source.list 文件：

$ sudo apt edit\-sources

1.搜索软件

sudo  apt\-cache  search  package\_name

其中还可以使用正则表达式 sudo apt\-cache search sof\* 这样就可以搜索到源上面所有以sof开头的软件包。

2.查看软件包信息

sudo apt\-cache show package\_name

3.查看软件包依赖关系

sudo apt\-cache show depends package\_name

4.查看每个软件包的简要信息

sudo apt\-cache dump

5.安装软件

sudo apt\-get install  package\_name

6.更新已安装的软件包

sudo apt\-get  upgrade

7.更新软件包列表

sudo apt\-get update

8.卸载一个软件包但是保留相关的配置文件

sudo apt\-get remove package\_name

9.卸载一个软件包同时删除配置文件

apt\-get \-purge remove package\_name

10.删除软件包的备份

apt\-get clean

参考资料：[https://www.cnblogs.com/sparkdev/p/11357343.html](https://www.cnblogs.com/sparkdev/p/11357343.html)
