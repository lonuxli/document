# 环境工具之Ubuntu下使用记录

[http://download.chinaunix.net/](http://download.chinaunix.net/)

[http://www.linuxfromscratch.org/blfs/view/svn/basicnet/rpcbind.html](http://www.linuxfromscratch.org/blfs/view/svn/basicnet/rpcbind.html)

**一、SSH服务搭建�**�

sudo apt\-get install openssh\-server  

ps \-e |grep ssh   运行ssh

**二、FTP服务搭建**

sudo apt\-get install vsftpd        安装vsftpd

sudo vim vsftpd.conf             配置ftp

sudo /etc//init.d/vsftpd  restart    重启ftp服务

## [ubuntu 修改资源镜像](https://www.cnblogs.com/scotth/p/6533397.html)

要修改的文件 /etc/apt/sources.list

原资源镜像文件

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) yakkety main multiverse restricted universe

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) yakkety\-backports main multiverse restricted universe

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) yakkety\-proposed main multiverse restricted universe

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) yakkety\-security main multiverse restricted universe

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) yakkety\-updates main multiverse restricted universe

deb\-src http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) yakkety main multiverse restricted universe

deb\-src http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) yakkety\-backports main multiverse restricted universe

deb\-src http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) yakkety\-proposed main multiverse restricted universe

deb\-src http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) yakkety\-security main multiverse restricted universe

deb\-src http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) yakkety\-updates main multiverse restricted universe

一个个改太麻烦，用脚本方便点

#### 163

Codename=$\( \(lsb\_release \-a\)|awk '{print $2}'|tail \-n 1 \)echo "\\deb [http://mirrors.163.com/ubuntu/](http://mirrors.163.com/ubuntu/)$Codename main multiverse restricted universedeb [http://mirrors.163.com/ubuntu/](http://mirrors.163.com/ubuntu/)$Codename\-backports main multiverse restricted universedeb [http://mirrors.163.com/ubuntu/](http://mirrors.163.com/ubuntu/)$Codename\-proposed main multiverse restricted universedeb [http://mirrors.163.com/ubuntu/](http://mirrors.163.com/ubuntu/)$Codename\-security main multiverse restricted universedeb [http://mirrors.163.com/ubuntu/](http://mirrors.163.com/ubuntu/)$Codename\-updates main multiverse restricted universedeb\-src [http://mirrors.163.com/ubuntu/](http://mirrors.163.com/ubuntu/)$Codename main multiverse restricted universedeb\-src [http://mirrors.163.com/ubuntu/](http://mirrors.163.com/ubuntu/)$Codename\-backports main multiverse restricted universedeb\-src [http://mirrors.163.com/ubuntu/](http://mirrors.163.com/ubuntu/)$Codename\-proposed main multiverse restricted universedeb\-src [http://mirrors.163.com/ubuntu/](http://mirrors.163.com/ubuntu/)$Codename\-security main multiverse restricted universedeb\-src [http://mirrors.163.com/ubuntu/](http://mirrors.163.com/ubuntu/)$Codename\-updates main multiverse restricted universe "\>sources.list

### ali yun

Codename=$\( \(lsb\_release \-a\)|awk '{print $2}'|tail \-n 1 \)echo "\\deb [http://mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/)$Codename main multiverse restricted universedeb [http://mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/)$Codename\-backports main multiverse restricted universedeb [http://mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/)$Codename\-proposed main multiverse restricted universedeb [http://mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/)$Codename\-security main multiverse restricted universedeb [http://mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/)$Codename\-updates main multiverse restricted universedeb\-src [http://mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/)$Codename main multiverse restricted universedeb\-src [http://mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/)$Codename\-backports main multiverse restricted universedeb\-src [http://mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/)$Codename\-proposed main multiverse restricted universedeb\-src [http://mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/)$Codename\-security main multiverse restricted universedeb\-src [http://mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/)$Codename\-updates main multiverse restricted universe "\>sources.list

类似把 [mirrors.aliyun.com](http://mirrors.aliyun.com/) 改成[mirrors.163.com](http://mirrors.163.com/) mirrors.utsc.cn 就可以

这个可以用：

deb\-src http://[archive.ubuntu.com/ubuntu](http://archive.ubuntu.com/ubuntu) xenial main restricted \#Added by software\-properties

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial main restricted

deb\-src http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial main restricted multiverse universe \#Added by software\-properties

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial\-updates main restricted

deb\-src http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial\-updates main restricted multiverse universe \#Added by software\-properties

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial universe

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial\-updates universe

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial multiverse

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial\-updates multiverse

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial\-backports main restricted universe multiverse

deb\-src http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial\-backports main restricted universe multiverse \#Added by software\-properties

deb http://[archive.canonical.com/ubuntu](http://archive.canonical.com/ubuntu) xenial partner

deb\-src http://[archive.canonical.com/ubuntu](http://archive.canonical.com/ubuntu) xenial partner

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial\-security main restricted

deb\-src http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial\-security main restricted multiverse universe \#Added by software\-properties

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial\-security universe

deb http://[mirrors.aliyun.com/ubuntu/](http://mirrors.aliyun.com/ubuntu/) xenial\-security multiverse

备注：以下是在非root用户下的配置，如果是在root用户下，把sudo 去掉即可。

1.安装samba：

1、安装samba软件包\#sudo apt\-get install samba\#sudo apt\-get install smbclient 

2、配置samba服务\#vi /etc/samba/smb.conf，编辑该文件。在Windows系统中不用输入密码访问Linux共享目录在Linux共享一个目录，将建立好的目录的设置信息写入/etc/smb.conf文件即可。如：若共享/home/shenrt目录，要在Windows系统中访问这个共享的目录，假设Windows主机的IP为192.168.2.100，Linux主机的IP为192.168.2.236，进行如下操作：\#mkdir /home/shenrt\#vi smb.conf 

    a、将文件中的内容做如下相应修改：security=user 改为security=share \(注意：要去掉\#，即不可以： \#security=share\)

    b、在文件结尾添加如下行：

\[share\]comment=this is Linux share directory

path=/home/shenrt

public=yes

writable=yes

3、重启samba服务

\#service smbd restart

\#service nmbd restart

4、测试是否可以访问linux共享目录     在windows xp下打开“我的电脑”，点击“工具”里的“映射网络驱动器”。     在弹出的窗口里的文件夹框里输入“\\\\192.168.2.236\\share”，并确定完成          \\\\192.168.119.133\\share

ubuntu 18.04 设置默认ip ：

修改配置文件：/etc/netplan/50\-cloud\-init.yaml

  6 network:

  7     ethernets:

  8         ens33:

  9             addresses: \[192.168.119.155/24, \]

10             gateway4: 192.168.119.2

11             nameservers:

12                 addresses: \[8.8.8.8\]

13     version: 2

NFS实现：

服务器端：sudo apt install nfs\-kernel\-server

sudo vim /etc/exports

/home \*\(rw,sync,no\_root\_squash\)

sudo /etc/init.d/nfs\-kernel\-server restart

sudo mount 192.168.119.155:/share/nshare  /share/nfs

在18.04 ubuntu 服务器上 运行qemu工具，代码编译在ubuntu 12.04上进行

cd /share/nshare

linux\-3.10.1:    

qemu\-system\-arm \-M vexpress\-a9 \-m 512M \-kernel zImage \-nographic \-append "root=/dev/mmcblk0  console=ttyAMA0" \-sd rootfs\-arm\-32m.ext3

linux4.9.94:

qemu\-system\-arm \-M vexpress\-a9 \-m 512M \-kernel zImage \-dtb vexpress\-v2p\-ca9.dtb \-nographic \-append "root=/dev/mmcblk0  console=ttyAMA0 debug memblock=debug" \-sd rootfs\-arm\-32m.ext3

1804server:ll/11126114

git clone git@192.168.119.155://share/linux\_git/mycode.git

gdb 调试实现：

qemu\-system\-arm \-M vexpress\-a9 \-m 512M \-kernel zImage \-nographic \-append "root=/dev/mmcblk0  console=ttyAMA0" \-sd     \-gdb tcp::1234 \-S

/share/nfs/gdb  /share/gitwork/mycode/linux\_kernel/linux\-3.10.1/vmlinux

target remote ： 1234
