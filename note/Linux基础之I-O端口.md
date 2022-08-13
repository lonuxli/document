# Linux基础之I/O端口

**Linux下的IO端口和IO内存**

 

CPU对外设端口物理地址的编址方式有两种：一种是IO映射方式，另一种是内存映射方式。 

　Linux将基于IO映射方式的和内存映射方式的IO端口统称为IO区域（IO region）。

　　IO region仍然是一种IO资源，因此它仍然可以用resource结构类型来描述。

     对I/O端口的访问

　　1\) request\_region\(\)

　　把一个给定区间的IO端口分配给一个IO设备。

　　2\) check\_region\(\)

　　检查一个给定区间的IO端口是否空闲，或者其中一些是否已经分配给某个IO设备。

　　3\) release\_region\(\)

　　释放以前分配给一个IO设备的给定区间的IO端口。

　　Linux中可以通过以下辅助函数来访问IO端口：

　　inb\(\),inw\(\),inl\(\),outb\(\),outw\(\),outl\(\)

　　“b”“w”“l”分别代表8位，16位，32位。

　对IO内存资源的访问

　　1\) request\_mem\_region\(\)

　　请求分配指定的IO内存资源。

　　2\) check\_mem\_region\(\)

　　检查指定的IO内存资源是否已被占用。

　　3\) release\_mem\_region\(\)

　　释放指定的IO内存资源。

**对外设的访问**

**1、访问I/O内存**的流程是：**request\_mem\_region\(\)** \-\>** ioremap\(\)** \-\> **ioread8\(\)/iowrite8\(\)** \-\> **iounmap\(\)** \-\>**release\_mem\_region\(\)�**�。

        前面说过，IO内存是统一编址下的概念，对于统一编址，IO地址空间是物理主存的一部分，对于编程而言，我们只能操作虚拟内存，所以，访问的第一步就是要把设备所处的物理地址映射到虚拟地址，Linux2.6下用ioremap\(\):

        void \*ioremap\(unsigned long offset, unsigned long size\);

然后，我们可以直接通过指针来访问这些地址，但是也可以用Linux内核的一组函数来读写：

ioread8\(\), iowrite16\(\), ioread8\_rep\(\), iowrite8\_rep\(\)......

**2、访问I/O端口**

        访问IO端口有2种途径：**I/O映射方式（I/O－mapped）、内存映射方式（Memory－mapped）。**前一种途径不映射到内存空间，直接使用 intb\(\)/outb\(\)之类的函数来读写IO端口；后一种MMIO是先把IO端口映射到IO内存（“内存空间”），再使用访问IO内存的函数来访问 IO端口。

        **void ioport\_map\(unsigned long port, unsigned int count\);**通过这个函数，可以把port开始的count个连续的IO端口
