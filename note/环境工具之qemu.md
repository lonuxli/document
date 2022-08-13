# 环境工具之qemu

# Ubuntu 18.04 使用QEMU搭建ARM Linux开发环境

一、安装工具：

    $ sudo apt\-get install vim openssh\-server net\-tools nfs\-kernel\-server git qemu crash \\

build\-essential libncurses\-dev libssl\-dev gcc\-arm\-linux\-gnueabi gcc\-aarch64\-linux\-gnu systemtap

二、下载源码：

    linux\-4.15.10

[https://cdn.kernel.org/pub/linux/kernel/v4.x/linux\-4.15.10.tar.gz](https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.15.10.tar.gz)

    busybox\-1.28.1

[http://busybox.net/downloads/busybox\-1.28.1.tar.bz2](http://busybox.net/downloads/busybox-1.28.1.tar.bz2)

  

三、制作根文件系统：

    $ cd busybox\-1.28.1

    $ export ARCH=arm

    $ export CROSS\_COMPILE=arm\-linux\-gnueabi\-

    $ make menuconfig

    $ make \-j8

    $ make install

    注意：make menuconfig时,

1. Settings  \-\-\-\> \[\*\] Build BusyBox as a static binary \(no shared libs\)
2. Settings  \-\-\-\>

    

    $ cd ../linux\-4.15.10/rootfs\_arm32

    $ mkdir dev sys proc etc tmp var mnt root

    $ cd dev

    $ sudo mknod console c 5 1

    $ sudo mknod tty1 c 4 1

    $ sudo mknod tty2 c 4 2

    $ sudo mknod tty3 c 4 3

    $ sudo mknod tty4 c 4 4

    $ sudo mknod null c 1 3

    $ cd ../etc

    $ mkdir init.d

    $ cd init.d

    $ vim rcS

    $ chmod 0755 rcS

    注意：rcS的内容为

\#\!/bin/sh

mount \-t proc proc /proc

mount \-t sysfs sysfs /sys

/sbin/mdev \-s

四、编译ARM32内核：

    $ export ARCH=arm

    $ export CROSS\_COMPILE=arm\-linux\-gnueabi\-

    $ make vexpress\_defconfig

    $ make menuconfig

    $ make bzImage \-j8

    $ make dtbs

  

    注意：make menuconfig时，

1. General setup  \-\-\-\> \(rootfs\_arm32\) Initramfs source file\(s\) 

即，CONFIG\_INITRAMFS\_SOURCE=“rootfs\_arm32"

   

五、运行ARM32内核：

    $ qemu\-system\-arm \-M vexpress\-a9 \-smp 2 \-m 1024M \-kernel arch/arm/boot/zImage  \-append "rdinit=/linuxrc console=ttyAMA0 loglevel=8” \-dtb arch/arm/boot/dts/vexpress\-v2p\-ca9.dtb \-nographic

六、制作ARM64根文件系统：

    $ cd busybox\-1.28.1

    $ export ARCH=arm64

    $ export CROSS\_COMPILE=aarch64\-linux\-gnu\-

    $ make menuconfig

    $ make \-j8

    $ make install

    注意：make menuconfig时,

1. Settings  \-\-\-\> \[\*\] Build BusyBox as a static binary \(no shared libs\)
2. Settings  \-\-\-\>

    

    $ cd ../linux\-4.15.10/rootfs\_arm64

    $ mkdir dev sys proc etc tmp var mnt root

    $ sudo cp \-r ../rootfs\_arm32/dev/\* rootfs\_arm64/dev/

    $ sudo cp \-r ../rootfs\_arm32/etc/\* rootfs\_arm64/etc/

七、编译ARM64内核：

    $ export ARCH=arm64

    $ export CROSS\_COMPILE=aarch64\-linux\-gnu\-

    $ make defconfig

    $ make menuconfig

    $ make \-j8

 

    注意：make menuconfig时，

1. General setup  \-\-\-\> \(rootfs\_arm64\) Initramfs source file\(s\) 

即，CONFIG\_INITRAMFS\_SOURCE="rootfs\_arm64” 

八、运行ARM64内核：

    $ qemu\-system\-aarch64 \-M virt \-cpu cortex\-a57 \-nographic \-m 2048M \-smp 2 \-kernel arch/arm64/boot/Image \-append "rdinit=/linuxrc console=ttyAMA0 loglevel=8" \-nographic

九、配置nfs:

    $ sudo mkdir /nfsroot

    $ sudo chmod 0777 /nfsroot

    $ sudo vim /etc/exports

    $ /etc/init.d/nfs\-kernel\-server restart

    在/etc/exports尾部加一行:

/nfsroot \*\(rw,sync,no\_root\_squash,no\_subtree\_check\)

mount \-t nfs \-o rw,tcp,nolock 192.168.119.129:/nfsroot /nfs

mount \-t cifs //192.168.119.129/share  /home \-o user=root,password=11126114

mount \-t cifs //192.168.119.129/home/lilong/share  /home \-o user=root,password=11126114

十、tftpd服务搭建

tftp服务器搭建步骤

1、安装tftp\-server

使用 sudo apt\-get install tftpd\-hpa 命令下载tftp服务端

使用 sudo apt\-get install tftp\-hpa 命令下载客户端

2、配置tftp服务器

使用 sudo vi /etc/default/tftpd\-hpa 命令将源文件改为：

TFTP\_USERNAME = "tftp"

TFTP\_DIRCTORY = "/root/tftpboot"

TFTP\_ADDRESS = "0.0.0.0:69"

TFTP\_OPTIONS = "\-l \-c \-s"

注意：在配置之前先使用mkdir /root/tftpboot 命令创建一个目录，使用chmod 777 /root/tftpboot命令修改该目录的权限

3、重启tftp服务

sudo service tftpd\-hpa restart 重启服务

sudo service tftpd\-hpa status 查看服务运行状态

[https://www.jianshu.com/p/91baa4d140a2](https://www.jianshu.com/p/91baa4d140a2)
