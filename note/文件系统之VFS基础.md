# 文件系统之VFS基础

**二、关键数据结构**

**2.1 vfsmount**

```
struct vfsmount {
    struct dentry *mnt_root;    /* root of the mounted tree */
    struct super_block *mnt_sb; /* pointer to superblock */
    int mnt_flags;
};
```

每个装载的文件系统都对应于一个vfsmount结构的实例，通过vfsmount搭建起文件系统中结构化的挂载关系，这种关系类似于dentry搭建起来的目录结构关系。

```
struct mount {
    struct hlist_node mnt_hash;
    struct mount *mnt_parent;
    struct dentry *mnt_mountpoint;
    struct vfsmount mnt;
    union {
        struct rcu_head mnt_rcu;
        struct llist_node mnt_llist;
    };
#ifdef CONFIG_SMP
    struct mnt_pcp __percpu *mnt_pcp;
#else
    int mnt_count;
    int mnt_writers;
#endif
    struct list_head mnt_mounts;    /* list of children, anchored here */
    struct list_head mnt_child; /* and going through their mnt_child */
    struct list_head mnt_instance;  /* mount instance on sb->s_mounts */ 
    const char *mnt_devname;    /* Name of device e.g. /dev/dsk/hda1 */
    struct list_head mnt_list;
    struct list_head mnt_expire;    /* link in fs-specific expiry list */
    struct list_head mnt_share; /* circular list of shared mounts */
    struct list_head mnt_slave_list;/* list of slave mounts */
    struct list_head mnt_slave; /* slave list entry */
    struct mount *mnt_master;   /* slave is on master->mnt_slave_list */
    struct mnt_namespace *mnt_ns;   /* containing namespace */
    struct mountpoint *mnt_mp;  /* where is it mounted */
    struct hlist_node mnt_mp_list;  /* list mounts with the same mountpoint */
    struct list_head mnt_umounting; /* list entry for umount propagation */
#ifdef CONFIG_FSNOTIFY
    struct hlist_head mnt_fsnotify_marks;
    __u32 mnt_fsnotify_mask;
#endif
    int mnt_id;         /* mount identifier */
    int mnt_group_id;       /* peer group identifier */
    int mnt_expiry_mark;        /* true if marked for expiry */
    struct hlist_head mnt_pins;
    struct fs_pin mnt_umount;
    struct dentry *mnt_ex_mountpoint;
};
```

**2.2 超级块super\_block**

```
struct super_block {
    struct list_head    s_list;     /* Keep this first */  所有超级块链表
    dev_t           s_dev;      /* search index; _not_ kdev_t */
    unsigned char       s_blocksize_bits;
    unsigned long       s_blocksize;
    loff_t          s_maxbytes; /* Max file size */
    struct file_system_type *s_type; 文件系统类型
    const struct super_operations   *s_op; 超级块方法
    const struct dquot_operations   *dq_op;
    const struct quotactl_ops   *s_qcop;
    const struct export_operations *s_export_op;
    unsigned long       s_flags;
    unsigned long       s_iflags;   /* internal SB_I_* flags */
    unsigned long       s_magic;
    struct dentry       *s_root;  //文件系统根目录的目录对象
    struct rw_semaphore s_umount;
    int         s_count;
    atomic_t        s_active;
    const struct xattr_handler **s_xattr;
    const struct fscrypt_operations *s_cop;
    struct hlist_bl_head    s_anon;     /* anonymous dentries for (nfs) exporting */
    struct list_head    s_mounts;   /* list of mounts; _not_ for fs use */
    struct block_device *s_bdev;
    struct backing_dev_info *s_bdi;
    struct mtd_info     *s_mtd;
    struct hlist_node   s_instances;  //相同的文件系统下超级块之间链表
    unsigned int        s_quota_types;  /* Bitmask of supported quota types */
    struct quota_info   s_dquot;    /* Diskquota specific options */
    struct sb_writers   s_writers;
    char s_id[32];              /* Informational name */
    u8 s_uuid[16];              /* UUID */
    void            *s_fs_info; /* Filesystem private info */
    unsigned int        s_max_links;
    fmode_t         s_mode;
    struct user_namespace *s_user_ns;
    struct list_lru     s_dentry_lru ____cacheline_aligned_in_smp;
    struct list_lru     s_inode_lru ____cacheline_aligned_in_smp;
    struct rcu_head     rcu;
    struct work_struct  destroy_work;
    struct mutex        s_sync_lock;    /* sync serialisation lock */
    int s_stack_depth;
    struct list_head    s_inodes;   /* all inodes */  超级块下所有的inodes
    spinlock_t      s_inode_wblist_lock;
    struct list_head    s_inodes_wb;    /* writeback inodes */ 需要回写的inode
};
```

super\_block保存了文件系统它本身和装载点的有关信息

**2.3 索引节点inode**

```
struct inode {
    umode_t         i_mode;
    unsigned short      i_opflags;
    kuid_t          i_uid;  //所有者标识符
    kgid_t          i_gid;   //组标识符
    unsigned int        i_flags;
    const struct inode_operations   *i_op;  //inode操作方法
    struct super_block  *i_sb;  //指向该inode的super_block
    struct address_space    *i_mapping;
    unsigned long       i_ino;
    union {
        const unsigned int i_nlink;
        unsigned int __i_nlink;
    };
    dev_t           i_rdev;
    loff_t          i_size;
    struct timespec     i_atime;
    struct timespec     i_mtime;
    struct timespec     i_ctime;
    spinlock_t      i_lock; /* i_blocks, i_bytes, maybe i_size */
    unsigned short          i_bytes
   unsigned int        i_blkbits;
    blkcnt_t        i_blocks;

    /* Misc */
    unsigned long       i_state;
    struct rw_semaphore i_rwsem;

    unsigned long       dirtied_when;   /* jiffies of first dirtying */
    unsigned long       dirtied_time_when;

    struct hlist_node   i_hash;
    struct list_head    i_io_list;  /* backing dev IO list */
    struct list_head    i_lru;      /* inode LRU list */
    struct list_head    i_sb_list;  超级块链表s_inodes连接
    struct list_head    i_wb_list;  /* backing dev writeback list */
    union {
        struct hlist_head   i_dentry;
        struct rcu_head     i_rcu;
    };
    u64         i_version;
    atomic_t        i_count;
    atomic_t        i_dio_count;
    atomic_t        i_writecount;
    const struct file_operations    *i_fop; /* former ->i_op->default_file_ops */
    struct file_lock_context    *i_flctx;
  struct address_space    i_data;
    struct list_head    i_devices;
    union {
        struct pipe_inode_info  *i_pipe;
        struct block_device *i_bdev;
        struct cdev     *i_cdev;
        char            *i_link;
        unsigned        i_dir_seq;
    };
    void            *i_private; /* fs or device private pointer */
};
```

inode的成员可能分为下面两类。

\(1\) 描述文件状态的元数据。例如，访问权限或上次修改的日期。

\(2\) 保存实际文件内容的数据段（或指向数据的指针）。就文本文件来说，用于保存文本，就目录而言，用户保存该目录下的目录项信息（所有文件或目录信息）。

查找起始于inode\(根目录 / 的inode\)，对系统来说必须总是已知的（根目录的dentry和inode是已知的）。该目录由一个inode表示，

其数据段并不包含普通数据，而是根目录下的各个目录项。这些项可能代表文件或其他目录。每个项由两个成员组成。

\(1\) 该目录项的数据所在inode的编号。（子目录或文件对应的inode编号）

\(2\) 文件或目录的名称。

1、对每个符号链接都使用了一个独立的inode。相应inode的数据段包含一个字符串，给出了链接目标的路径。

2、对于符号链接，可以区分原始文件和链接。对于硬链接，情况不是这样。在硬链接已经建立后，

无法区分哪个文件是原来的，哪个是后来建立的。在硬链接建立时，创建的目录项使用了一个现存的inode编号。

**2.4 目录缓存dentry**

```
struct dentry {
    /* RCU lookup touched fields */
    unsigned int d_flags;       /* protected by d_lock */
    seqcount_t d_seq;       /* per dentry seqlock */
    struct hlist_bl_node d_hash;    /* lookup hash list */
    struct dentry *d_parent;    /* parent directory */
    struct qstr d_name;
    struct inode *d_inode;      /* Where the name belongs to - NULL is negative */
    unsigned char d_iname[DNAME_INLINE_LEN];    /* small names */

    /* Ref lookup also touches following */
    struct lockref d_lockref;   /* per-dentry lock and refcount */
    const struct dentry_operations *d_op;
    struct super_block *d_sb;   /* The root of the dentry tree */
    unsigned long d_time;       /* used by d_revalidate */
    void *d_fsdata;         /* fs-specific data */

    union {
        struct list_head d_lru;     /* LRU list */
        wait_queue_head_t *d_wait;  /* in-lookup ones only */
    };
    struct list_head d_child;   /* child of parent list */
    struct list_head d_subdirs; /* our children */
    /*
     * d_alias and d_rcu can share memory
     */
    union {
        struct hlist_node d_alias;  /* inode alias list */
        struct hlist_bl_node d_in_lookup_hash;  /* only for in-lookup ones */
        struct rcu_head d_rcu;
    } d_u;
};
```

dentry的来源： dentry结构的主要用途是建立文件名和相关的inode之间的关联

1、由于块设备速度较慢，可能需要很长时间才能找到与一个文件名关联的inode。即使设备数据已经在页缓存中，仍然每次都会重复整个查找操作。

2、Linux使用目录项缓存（简称dentry缓存）来快速访问此前的查找操作的结果。

3、vfs中本可以使用inode来建立目录结构信息（inode目录节点和inode文件节点），但是这种建立可能比较耗时，利用dentry缓存来建立这种结构关系利于提高性能。

4、dentry只是一种内存缓存结构信息，在磁盘中无相应的数据。

     

2.6 address\_space

**2.7 fs\_struct**

fs\_struct 描述进程当前目录根目录（文件系统特定于进程的信息）

```
struct fs_struct {
    int users;
    spinlock_t lock;
    seqcount_t seq;
    int umask;
    int in_exec;
    struct path root, pwd;
};

//vfsmount和dentry两者可以定位文件的路径
struct path {
        struct vfsmount *mnt;
        struct dentry *dentry;
};
```

**2.8 files\_struct** 

files\_struct 描述进程打开的文件情况

```
struct files_struct {
  /*
   * read mostly part
   */
    atomic_t count;
    bool resize_in_progress;
    wait_queue_head_t resize_wait;
/*
为什么有两个fdtable呢？这是内核的一种优化策略。fdt为指针，而fdtab为普通变量。一般情况下， fdt是指向fdtab的，当需要它的时候，才会真正动态申请内存。因为默认大小的文件表足以应付大多数情况，因此这样就可以避免频繁的内存申请。这也是内核的常用技巧之一。在创建时，使用普通的变量或者数组，然后让指针指向它，作为默认情况使用。只有当进程使用量超过默认值时，才会动态申请内存。
*/
    struct fdtable __rcu *fdt;
    struct fdtable fdtab;
  /*
   * written part on a separate cache line in SMP
   */
    spinlock_t file_lock ____cacheline_aligned_in_smp;
    unsigned int next_fd;
    unsigned long close_on_exec_init[1];
    unsigned long open_fds_init[1];
    unsigned long full_fds_bits_init[1];
    struct file __rcu * fd_array[NR_OPEN_DEFAULT];
};

struct fdtable {
    unsigned int max_fds;
    struct file __rcu **fd;      /* current fd array */
    unsigned long *close_on_exec;
    unsigned long *open_fds;
    unsigned long *full_fds_bits;
    struct rcu_head rcu;
};
```

**2.8.1 files\_struct初始状态**

初始情况下，进程的files\_struct初始状态可以参考init\_files的定义，理清其用法：files\_struct的成员fdt在这里初始化为其内嵌的struct fdtable变量实例\(成员fdtab\)地址，内嵌的struct fdtable实例变量的fd，close\_on\_exec，open\_fds又指向属主的相应变量。

```
struct files_struct init_files = {
        .count          = ATOMIC_INIT(1),
        .fdt            = &init_files.fdtab,
        .fdtab          = {
                .max_fds        = NR_OPEN_DEFAULT,
                .fd             = &init_files.fd_array[0],
                .close_on_exec  = init_files.close_on_exec_init,
                .open_fds       = init_files.open_fds_init,
                .full_fds_bits  = init_files.full_fds_bits_init,
        },
        .file_lock      = __SPIN_LOCK_UNLOCKED(init_files.file_lock),
        .resize_wait    = __WAIT_QUEUE_HEAD_INITIALIZER(init_files.resize_wait),
};     
```

![files_struct_init.png](image/files_struct_init.png)

**2.8.2 files\_struct扩�**��

进行struct files\_struct扩充时，会分配一个新的struct fdtable。另外还分配了满足扩充要求的fd数组\(即struct file数组），以及与fd相对的bitmap描述close\_on\_exec，open\_fds的存储空间。然后将新分配的close\_on\_exec，open\_fds，fd空间指针赋值给new\_fdt\-\>close\_on\_exec，new\_fdt\-\>open\_fds和new\_fdt\-\>fd。注意，这里的close\_on\_exec，open\_fds和上面初始化时close\_on\_exec\_init，open\_fds\_init的差别：

```
static int expand_fdtable(struct files_struct *files, unsigned int nr)
        __releases(files->file_lock)
        __acquires(files->file_lock)
{
        struct fdtable *new_fdt, *cur_fdt;

        spin_unlock(&files->file_lock);
        new_fdt = alloc_fdtable(nr);  //先分配目标大小new_fdt

        spin_lock(&files->file_lock);
        cur_fdt = files_fdtable(files);
        BUG_ON(nr < cur_fdt->max_fds);
        copy_fdtable(new_fdt, cur_fdt);  //将旧的fdt复制到新的且更大的fdt中
        rcu_assign_pointer(files->fdt, new_fdt);  //利用rcu发布使得files_struct指向新申请的fdt

        //回收旧的fdt，只有经过扩展申请的就fdt才需要通过rcu释放。files->fdtab 不需要释放，一方面因为
        //其数据结构本身存放在files_struct中，另一方面其通过fd_array指向的struct file在扩展复制时被新的fdt指向了继续使用。
        if (cur_fdt != &files->fdtab)
                call_rcu(&cur_fdt->rcu, free_fdtable_rcu);  
        /* coupled with smp_rmb() in __fd_install() */
        smp_wmb();
        return 1;
}
```

扩充替换如下图所示：将旧的

![files_struct_expand.png](image/files_struct_expand.png)

**2.9 打开的文件对象file**

```
struct file {
    union {
        struct llist_node   fu_llist;
        struct rcu_head     fu_rcuhead;
    } f_u;
    struct path     f_path;
    struct inode        *f_inode;   /* cached value */
    const struct file_operations    *f_op;

    /*
     * Protects f_ep_links, f_flags.
     * Must not be taken from IRQ context.
     */
    spinlock_t      f_lock;
    atomic_long_t       f_count;
    unsigned int        f_flags;
    fmode_t         f_mode;
    struct mutex        f_pos_lock;
    loff_t          f_pos;
    struct fown_struct  f_owner;
    const struct cred   *f_cred;
    struct file_ra_state    f_ra; //文件预读参数

    u64         f_version;
#ifdef CONFIG_SECURITY
    void            *f_security;
#endif
    /* needed for tty driver, and maybe others */
    void            *private_data;

#ifdef CONFIG_EPOLL
    /* Used by fs/eventpoll.c to link all the hooks to this file */
    struct list_head    f_ep_links;
    struct list_head    f_tfile_llink;
#endif /* #ifdef CONFIG_EPOLL */
    struct address_space    *f_mapping;
} __attribute__((aligned(4)));  /* lest something weird decides that 2 is OK */
```

三、文件关系结构框架

![d1da07bd290a782bab976f92e97d2209.png](image/d1da07bd290a782bab976f92e97d2209.png)

![e2dcfd22d4ebe1223808b2cd3276dbda.png](image/e2dcfd22d4ebe1223808b2cd3276dbda.png)
