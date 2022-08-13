# 环境工具之Git 服务器搭建

# Git 服务器搭建

上一章节中我们远程仓库使用了 Github，Github 公开的项目是免费的，但是如果你不想让其他人看到你的项目就需要收费。

这时我们就需要自己搭建一台Git服务器作为私有仓库使用。

接下来我们将以 Centos 为例搭建 Git 服务器。

### 1、安装Git

$ yum install curl\-devel expat\-devel gettext\-devel openssl\-devel zlib\-devel perl\-devel

$ yum install git

接下来我们 创建一个git用户组和用户，用来运行git服务：

$ groupadd git

$ useradd git \-g git

### 2、创建证书登录

收集所有需要登录的用户的公钥，公钥位于id\_rsa.pub文件中，把我们的公钥导入到/home/git/.ssh/authorized\_keys文件里，一行一个。

如果没有该文件创建它：

$ cd /home/git/

$ mkdir .ssh

$ chmod 755 .ssh

$ touch .ssh/authorized\_keys

$ chmod 644 .ssh/authorized\_keys

### 3、初始化Git仓库

首先我们选定一个目录作为Git仓库，假定是/home/gitrepo/runoob.git，在/home/gitrepo目录下输入命令：

$ cd /home

$ mkdir gitrepo

$ chown git:git gitrepo/

$ cd gitrepo

$ git init \-\-bare runoob.git

Initialized empty Git repository in /home/gitrepo/runoob.git/

以上命令Git创建一个空仓库，服务器上的Git仓库通常都以.git结尾。然后，把仓库所属用户改为git：

$ chown \-R git:git runoob.git

### 4、克隆仓库

$ git clone git@192.168.45.4:/home/gitrepo/runoob.git

Cloning into 'runoob'...

warning: You appear to have cloned an empty repository.Checking connectivity... done.

192.168.45.4 为 Git 所在服务器 ip ，你需要将其修改为你自己的 Git 服务 ip。

这样我们的 Git 服务器安装就完成。

**修改git默认的编辑器：**

1\) 方法一 修改系统的配置

git config \-\-global core.editor vim

2\) 方法二 针对 git 项目修改

打开文件 .git/config

在 core 中添加editor=vim

git使用命令：

$ git branch dev 创建分支

$ git checkout dev    取出分支

$ git merge dev  合并指定分支到当前分支，

$ git branch \-d dev  删除分支dev

git删除文件

1、rm rootfs\_glibc\_master

2、git status 查看删除文件

3、git rm rootfs\_glibc\_master \-r 用git删除

4、git commit 提交删除
