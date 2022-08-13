# 虚拟化之namespace

Docker这么火，喜欢技术的朋友可能也会想，如果要自己实现一个资源隔离的容器，应该从哪些方面下手呢？也许你第一反应可能就是chroot命令，这条命令给用户最直观的感觉就是使用后根目录/的挂载点切换了，即文件系统被隔离了。然后，为了在分布式的环境下进行通信和定位，容器必然需要一个独立的IP、端口、路由等等，自然就想到了网络的隔离。同时，你的容器还需要一个独立的主机名以便在网络中标识自己。想到网络，顺其自然就想到通信，也就想到了进程间通信的隔离。可能你也想到了权限的问题，对用户和用户组的隔离就实现了用户权限的隔离。最后，运行在容器中的应用需要有自己的PID,自然也需要与宿主机中的PID进行隔离。

由此，我们基本上完成了一个容器所需要做的六项隔离，Linux内核中就提供了这六种namespace隔离的系统调用，如下表所示。

|Namespace|系统调用参数  |隔离内容                  |
|---------|--------------|--------------------------|
|UTS      |CLONE\_NEWUTS |主机名与域名              |
|IPC      |CLONE\_NEWIPC |信号量、消息队列和共享内存|
|PID      |CLONE\_NEWPID |进程编号                  |
|Network  |CLONE\_NEWNET |网络设备、网络栈、端口等等|
|Mount    |CLONE\_NEWNS  |挂载点（文件系统）        |
|User     |CLONE\_NEWUSER|用户和用户组              |

表 namespace六项隔离

实际上，Linux内核实现namespace的主要目的就是为了实现轻量级虚拟化（容器）服务。在同一个namespace下的进程可以感知彼此的变化，而对外界的进程一无所知。这样就可以让容器中的进程产生错觉，仿佛自己置身于一个独立的系统环境中，以此达到独立和隔离的目的。

命名空间是对全局作用域的细分。

随着大数据、虚拟化的兴起，Linux为了提供更加精细的资源分配管理机制，给出了namespace机制解决方法。

struct nsproxy {

    atomic\_t count;

    struct uts\_namespace \*uts\_ns;

    struct ipc\_namespace \*ipc\_ns;

    struct mnt\_namespace \*mnt\_ns;

    struct pid\_namespace \*pid\_ns;

    struct net          \*net\_ns;

};

初步理解：命名空间可以看作是task的数据环境集合，不同进程间按规则分类处于不同的环境
