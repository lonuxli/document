# 安全管理之user namespace

**一、背景**

User namespace 是其中最核心最复杂的，因为 user ns 是用来隔离和分割管理权限的，管理权限实质分为两部分 uid/gid 和 capability。

操作 namespace 的相关系统调用有 3 个：

-  fork\(\): 创建一个新的进程并把他放到新的 namespace 中。
-  setns\(\): 将当前进程加入到已有的 namespace 中。
-  unshare\(\): 使当前进程退出指定类型的 namespace，并加入到新创建的 namespace（相当于创建并加入新的 namespace）。

我们通常使用 unshare 命令来调用上述的各种函数。

注意：在创建新的 user namespace 时不需要任何权限；而在创建其他类型的 namespace\(UTS/PID/Mount/IPC/Network/Cgroup\)时，需要进程在对应 user namespace 中有CAP\_SYS\_ADMIN权限。

**二、数据结构**

struct user\_namespace表示一个user ns命名空间，在内核中构成树形结构，每个客体资源都应该有指向的user namespace。

```
struct user_namespace {
        struct uid_gid_map      uid_map;
        struct uid_gid_map      gid_map;
        struct uid_gid_map      projid_map;
        atomic_t                count;
        struct user_namespace   *parent;
        int                     level;
        kuid_t                  owner;
        kgid_t                  group;
        struct ns_common        ns;
        unsigned long           flags;
        struct ucounts          *ucounts;
        int ucount_max[UCOUNT_COUNTS];
};
```

uid\_map、gid\_map：用于建立父user ns与子user ns之间，id的映射关系

parent：构建树形的user ns结构，其中最顶层的是init\_user\_ns

level：表明user ns所在树中的层级

struct uid\_gid\_map表示当前user ns中局部id与全局id的映射关系

```
struct uid_gid_map {    /* 64 bytes -- 1 cache line */
        u32 nr_extents;
        struct uid_gid_extent {
                u32 first;
                u32 lower_first;
                u32 count;
        } extent[UID_GID_MAP_MAX_EXTENTS];
};
```

nr\_extents：表示映射的条目数量

first：某一映射条目中局部id起始值

lower\_first：某一映射条目中全局id起始值

count：某一映射条目中映射的id数量

```
struct cred {
        ...
        kuid_t          uid;            /* real UID of the task */
        kgid_t          gid;            /* real GID of the task */
        kuid_t          suid;           /* saved UID of the task */
        kgid_t          sgid;           /* saved GID of the task */
        kuid_t          euid;           /* effective UID of the task */
        kgid_t          egid;           /* effective GID of the task */
        kuid_t          fsuid;          /* UID for VFS ops */
        kgid_t          fsgid;          /* GID for VFS ops */
        unsigned        securebits;     /* SUID-less security management */
        kernel_cap_t    cap_inheritable; /* caps our children can inherit */
        kernel_cap_t    cap_permitted;  /* caps we're permitted */
        kernel_cap_t    cap_effective;  /* caps we can actually use */
        kernel_cap_t    cap_bset;       /* capability bounding set */
        kernel_cap_t    cap_ambient;    /* Ambient capability set */
        struct user_namespace *user_ns; /* user_ns the caps and keyrings are relative to. */
};

struct task_struct {
        ...
        const struct cred __rcu *real_cred; /* objective and real subjective task
                                         * credentials (COW) */
        const struct cred __rcu *cred;  /* effective (overridable) subjective task
                                         * credentials (COW) */
        struct nsproxy *nsproxy;
};

struct file {
    ...
    const struct cred       *f_cred;
};
```

![52adfdd6ccbce37047d97f75227e5ac5.png](image/52adfdd6ccbce37047d97f75227e5ac5.png)

**三、ID隔离**

**3.1 ID映射**

**3.1.1 映射建立规则**

创建完新的user namespace后，通过将用户映射信息写入该**user namespace中的其中一个进程对应**的/proc/ PID /uid\_map和 /proc/ PID /gid\_map文件来完成建立用户映射关系，写入格式如下：

```
ID-inside-ns   ID-outside-ns   length
```

**ID\-inside\-ns：**表明该段映射在当前被设置的user namespace内部ID起始值

**length：**表明该段映射的ID长度范围

**ID\-outside\-ns  ：**表示外部user namespace范围的起点ID。ID\-outside\-ns的解释方式取决于**打开文件（注意此处是打开，打开意味着是读或写）**/proc/ PID /uid\_map （或/proc/ PID /gid\_map ）的进程是否与进程PID位于同一用户命名空间中：

- 如果两个进程在同一个命名空间中。

           则ID\-outside\-ns 被解释为进程PID的父user namespace中的用户 ID（组 ID） 。这里的常见情况是进程正在写入自己的映射文件（/proc/self/uid\_map或 /proc/self/gid\_map）。

- 如果两个进程位于不同的命名空间中

           则 ID\-outside\-ns被解释为打开/proc/ PID /uid\_map \( /proc/ PID /gid\_map \)的进程的用户命名空间中的用户 ID（组 ID ）。由于非父子用户命名空间的进程间不能设置映射关系，因此如果两个用户命名空间不是父子关系则只能是读操作。

**3.1.2 映射写入的规则**

建立父子命名空间ID映射关系，即写/proc/ PID/uid\_map\( /proc/ PID /gid\_map \)，是比较苛刻的，规则具体如下

1、单个uid/gid\_map文件可写入的映射关系限制在5。后续是否增加可细看内核实现。

2、**uid/gid\_map文件归创建user namespace的用户ID所拥有，并且只能由该用户（或特权用户）写入。**即只有user namespace所属用户或特权用户才能设置映射关系。其文件权限如下：

```
-rw-r--r-- 1 lilong lilong 0 Mar 19 02:58 uid_map
-rw-r--r-- 1 lilong lilong 0 Mar 19 02:58 gid_map
```

3、**写入进程必须在被写进程PID的用户命名空间中具有CAP\_SETUID（用于gid\_map 的CAP\_SETGID ）能力**。

4、**写入进程必须位于被写进程PID的用户命名空间或****直接父用户命名空间�**�（ inside the \(immediate\) parent user namespace of the process _PID_.）反过来讲写入进程只能对其所属用户命名空间及其子用户命名空间设置映射关系。

5、以下两种情况，至少有一个必须满足。

- 仅仅只映射写入进程的有效用户ID。
- 写入进程在父命名空间中拥有

**3.1.3 映射查看的规则**

通过读取/proc/ PID /uid\_map和 /proc/ PID /gid\_map文件可以查看ID映射关系

其中ID\-outside\-ns 是根据打开文件的进程进行解释：

- 如果打开文件的进程与进程PID位于相同的用户命名空间中，则ID\-outside\-ns是相对于父用户命名空间定义的。
- 如果打开文件的进程在不同的用户命名空间中，则ID\-outside\-ns是相对于打开文件的进程的用户命名空间定义的

**3.2 局部ID和全局ID转换代码**

获取当前进程的uid/gid    task\-\>cred\-\>uid

```
#define current_uid()           (current_cred_xxx(uid))
#define current_cred_xxx(xxx)                   \
({                                              \
        current_cred()->xxx;                    \
})
#define current_cred() \
        rcu_dereference_protected(current->cred, 1)
```

获取task的uns是从task\-\>cred\-\>user\_ns中获取

例如：获取当前进程的uns

```
#define current_user_ns()       (current_cred_xxx(user_ns))
#define current_cred() \
        rcu_dereference_protected(current->cred, 1)
```

**3.2.1、全局uid\+user\_namespace转变成局部uid**

from\_kuid\(\)函数实现此功能，user\_namespace\-\>uid\_map保存了全局id与局部id的映射关系。from\_kgid\(\)类似实现gid转换功能。

```
uid_t from_kuid(struct user_namespace *targ, kuid_t kuid)
{
        /* Map the uid from a global kernel uid */
        return map_id_up(&targ->uid_map, __kuid_val(kuid));
}

static u32 map_id_up(struct uid_gid_map *map, u32 id)
{
        unsigned idx, extents;
        u32 first, last;

        /* Find the matching extent */
        extents = map->nr_extents;
        smp_rmb();
        for (idx = 0; idx < extents; idx++) {
                first = map->extent[idx].lower_first; //lower_first中保存的是全局id起始值
                last = first + map->extent[idx].count - 1;
                if (id >= first && id <= last)  //如果全局id在lower_first 与lower_first+count之间，说明存在映射关系
                        break;
        }
        /* Map the id or note failure */
        if (idx < extents)
                id = (id - first) + map->extent[idx].first;  //first保存局部id起始值，通过相对偏移计算出局部id值
        else 
                id = (u32) -1;

        return id;
}
```

**3.2.2、局部id\+user\_namespace转变成全局id**

make\_kuid\(\)实现此功能，相当于from\_kuid\(\)函数的一个逆向行为。make\_kgid\(\)类似实现gid的转换。

```
kuid_t make_kuid(struct user_namespace *ns, uid_t uid)
{
        /* Map the uid to a global kernel uid */
        return KUIDT_INIT(map_id_down(&ns->uid_map, uid));
}

static u32 map_id_down(struct uid_gid_map *map, u32 id)
{
        unsigned idx, extents;
        u32 first, last;

        /* Find the matching extent */
        extents = map->nr_extents;
        smp_rmb();
        for (idx = 0; idx < extents; idx++) {
                first = map->extent[idx].first; //first保存局部id起始值
                last = first + map->extent[idx].count - 1;
                if (id >= first && id <= last) //如果局部id在first 与first+count之间，说明存在映射关系
                        break;
        }
        /* Map the id or note failure */
        if (idx < extents)
                id = (id - first) + map->extent[idx].lower_first; //lower_first保存局部id起始值，通过相对偏移计算出局部id值
        else
                id = (u32) -1;

        return id;
}
```

**3.3 ID映射写入代码分析**

写入时写入进程的uns要么和被写入uns是相同的，要么前者是后者的父亲，不论哪一种ID\-outside\-ns都是父uns中局部id值。

```
ssize_t proc_uid_map_write(struct file *file, const char __user *buf,
                           size_t size, loff_t *ppos)
{
        struct seq_file *seq = file->private_data;
        struct user_namespace *ns = seq->private;  //文件打开时proc_id_map_open将被设置task的uns放进file->seq->private
        struct user_namespace *seq_ns = seq_user_ns(seq);

        if (!ns->parent)  //最顶层的根user namespace不能被设置
                return -EPERM;

        if ((seq_ns != ns) && (seq_ns != ns->parent)) //设置进程要么和被设置进程在同一uns，要么是其父uns。符合2.2节第4条判读规则。
                return -EPERM;

        return map_write(file, buf, size, ppos, CAP_SETUID,
                         &ns->uid_map, &ns->parent->uid_map);
}

static ssize_t map_write(struct file *file, const char __user *buf,
                         size_t count, loff_t *ppos,
                         int cap_setid,
                         struct uid_gid_map *map,
                         struct uid_gid_map *parent_map)
{
        struct seq_file *seq = file->private_data;
        struct user_namespace *ns = seq->private;
        struct uid_gid_map new_map;
        unsigned idx;
        struct uid_gid_extent *extent = NULL;
        char *kbuf = NULL, *pos, *next_line;
        ssize_t ret = -EINVAL;

        ret = -EPERM;
        /* Only allow one successful write to the map */  //文件设置过了就不能再次设置
        if (map->nr_extents != 0)
                goto out;
        //?
        if (cap_valid(cap_setid) && !file_ns_capable(file, ns, CAP_SYS_ADMIN)) //权限检查
                goto out;

        //循环解析字符串设置填充，此时lower_first中存的的还是父uns局部id
        for(; pos; pos = next_line) {
            ...
            extent->first = simple_strtoul(pos, &pos, 10);
            extent->lower_first = simple_strtoul(pos, &pos, 10);
            extent->count = simple_strtoul(pos, &pos, 10);
            new_map.nr_extents++;
        }

        if (new_map.nr_extents == 0)
                goto out;

        if (!new_idmap_permitted(file, ns, cap_setid, &new_map))  //权限检查
                goto out;

        //将父uns局部id全部转换成全局的id，所有的映射map中lower_first都是存的全局id
        for (idx = 0; idx < new_map.nr_extents; idx++) {
                u32 lower_first;
                extent = &new_map.extent[idx];

                lower_first = map_id_range_down(parent_map,
                                                extent->lower_first,
                                                extent->count);

                if (lower_first == (u32) -1)
                        goto out;

                extent->lower_first = lower_first;
        }

        /* Install the map */
        memcpy(map->extent, new_map.extent,
                new_map.nr_extents*sizeof(new_map.extent[0]));
        smp_wmb();
        map->nr_extents = new_map.nr_extents;

        *ppos = count;
        ret = count;
out:
        mutex_unlock(&userns_state_mutex);
        kfree(kbuf);
        return ret;
}

//调用new_idmap_permitted时，lower_first中存的还是局部id
static bool new_idmap_permitted(const struct file *file,
                                struct user_namespace *ns, int cap_setid,
                                struct uid_gid_map *new_map)
{
        const struct cred *cred = file->f_cred;

        if ((new_map->nr_extents == 1) && (new_map->extent[0].count == 1) &&
            uid_eq(ns->owner, cred->euid)) {  //如果只有一条映射并且设置者和ns的创建者的是同一个id
                u32 id = new_map->extent[0].lower_first;
                if (cap_setid == CAP_SETUID) {
                        kuid_t uid = make_kuid(ns->parent, id);
                        if (uid_eq(uid, cred->euid))
                                return true;  //如果只有一条映射并且是对设置者id的映射，则不检查权限，符合2.2节5中的第一个条件
                } else if (cap_setid == CAP_SETGID) {
                        kgid_t gid = make_kgid(ns->parent, id);
                        if (!(ns->flags & USERNS_SETGROUPS_ALLOWED) &&
                            gid_eq(gid, cred->egid))
                                return true;
                }
        }

        /* Allow anyone to set a mapping that doesn't require privilege */
        if (!cap_valid(cap_setid))
                return true;

        /* Allow the specified ids if we have the appropriate capability
         * (CAP_SETUID or CAP_SETGID) over the parent user namespace.
         * And the opener of the id file also had the approprpiate capability.
         */
        //ns_capable(ns->parent, cap_setid)判断当前进程在父uns中是否具备CAP_SETUID的权限
        //file_ns_capable(file, ns->parent, cap_setid)判断文件的打开者在父uns中是否具备CAP_SETUID的权限，这两者不是同一个？？
        if (ns_capable(ns->parent, cap_setid) &&
            file_ns_capable(file, ns->parent, cap_setid)) 
                return true;

        return false;
}
```

**3.4 ID映射读取代码**

读取/proc/pid/uid\_map时调用proc\_uid\_seq\_operations中的uid\_m\_show\(\)

```
static int uid_m_show(struct seq_file *seq, void *v)
{
        struct user_namespace *ns = seq->private;
        struct uid_gid_extent *extent = v;
        struct user_namespace *lower_ns;
        uid_t lower;

        lower_ns = seq_user_ns(seq);  //lower_ns是读取进程的uns
        if ((lower_ns == ns) && lower_ns->parent)
                lower_ns = lower_ns->parent;

        lower = from_kuid(lower_ns, KUIDT_INIT(extent->lower_first)); //将全局uid转成读取进程uns中的局部id。符合2.3节规则

        seq_printf(seq, "%10u %10u %10u\n",
                extent->first,
                lower,
                extent->count);

        return 0;
}
```

**3.5不同视角下的uid/gid**

**3.5.1 进程调用getuid\(\)系统调用获取自身uid**

这种情况是返回进程所在uns中局部id值

```
SYSCALL_DEFINE0(getuid)
{
        /* Only we change this so SMP safe */
        return from_kuid_munged(current_user_ns(), current_uid()); //current_uid() => task->cred->uid
}
uid_t from_kuid_munged(struct user_namespace *targ, kuid_t kuid)
{
        uid_t uid;
        uid = from_kuid(targ, kuid); //将全局id值转成局部id值

        if (uid == (uid_t) -1)
                uid = overflowuid;
        return uid;
}
```

**3.5.2 进程获取某个文件的uid/gid**

这种情况下看到的是文件的全局id在当前获取进程uns中映射的局部id值，如ls \-l命令查看文件时显式的是局部id，系统调用为stat

```
SYSCALL_DEFINE2(stat64, const char __user *, filename,
                struct stat64 __user *, statbuf)
{
        struct kstat stat;
        int error = vfs_stat(filename, &stat);

        if (!error)
                error = cp_new_stat64(&stat, statbuf);

        return error;
}    
static long cp_new_stat64(struct kstat *stat, struct stat64 __user *statbuf)
{
        struct stat64 tmp;

        INIT_STRUCT_STAT64_PADDING(tmp);

        tmp.st_dev = huge_encode_dev(stat->dev);
        tmp.st_rdev = huge_encode_dev(stat->rdev);
        tmp.st_ino = stat->ino;
        if (sizeof(tmp.st_ino) < sizeof(stat->ino) && tmp.st_ino != stat->ino)
                return -EOVERFLOW;
        tmp.st_mode = stat->mode;
        tmp.st_nlink = stat->nlink;
        tmp.st_uid = from_kuid_munged(current_user_ns(), stat->uid);
        tmp.st_gid = from_kgid_munged(current_user_ns(), stat->gid);
        tmp.st_atime = stat->atime.tv_sec;
        tmp.st_atime_nsec = stat->atime.tv_nsec;
        tmp.st_mtime = stat->mtime.tv_sec;
        tmp.st_mtime_nsec = stat->mtime.tv_nsec;
        tmp.st_ctime = stat->ctime.tv_sec;
        tmp.st_ctime_nsec = stat->ctime.tv_nsec;
        tmp.st_size = stat->size;
        tmp.st_blocks = stat->blocks;
        tmp.st_blksize = stat->blksize;
        return copy_to_user(statbuf,&tmp,sizeof(tmp)) ? -EFAULT : 0;
}
```

**3.6 ID使用判断**

**3.6.1 进程操作文件**

inode\_permission\(\) → \_\_inode\_permission\(\) → do\_inode\_permission\(\) → generic\_permission\(\) → acl\_permission\_check\(\)

```
int generic_permission(struct inode *inode, int mask)
{
        int ret;

        /*
         * Do the basic permission checks.
         */
        ret = acl_permission_check(inode, mask);
        if (ret != -EACCES)
                return ret;

        if (S_ISDIR(inode->i_mode)) {
                /* DACs are overridable for directories */
                if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE))
                        return 0;
                if (!(mask & MAY_WRITE))
                        if (capable_wrt_inode_uidgid(inode,
                                                     CAP_DAC_READ_SEARCH))
                                return 0;
                return -EACCES;
        }
        /*
         * Read/write DACs are always overridable.
         * Executable DACs are overridable when there is
         * at least one exec bit set.
         */
        if (!(mask & MAY_EXEC) || (inode->i_mode & S_IXUGO))
                if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE))
                        return 0;

        /*
         * Searching includes executable on directories, else just read.
         */
        mask &= MAY_READ | MAY_WRITE | MAY_EXEC;
        if (mask == MAY_READ)
                if (capable_wrt_inode_uidgid(inode, CAP_DAC_READ_SEARCH))
                        return 0;

        return -EACCES;
}
```

**3.6.2 进程操作进程**

以发送kill信号为例（ 其实主要还是capability权限判断）：

kill\_something\_info\(sig, &info, pid\)\-\>group\_send\_sig\_info\(\)\-\>check\_kill\_permission\(\)\-\>kill\_ok\_by\_cred\(\)

```
//返回1表明具备kill权限
static int kill_ok_by_cred(struct task_struct *t)
{
        const struct cred *cred = current_cred(); //操作进程（即代码所处上下文的进程）的凭证,task->cred  ,subjective credentials
        const struct cred *tcred = __task_cred(t); //被操作进程的凭证,task->real_cred ,objective credentials

        //判断发送信号进程（主体）在接收信号进程（客体）的"ID"是否相同，相同也可以发送信号
        if (uid_eq(cred->euid, tcred->suid) ||
            uid_eq(cred->euid, tcred->uid)  ||
            uid_eq(cred->uid,  tcred->suid) ||
            uid_eq(cred->uid,  tcred->uid))
                return 1;

        //判断发送信号进程（主体）在接收信号进程（客体）的uns中是否具备CAP_KILL权限，如果有则可以发送信号
        if (ns_capable(tcred->user_ns, CAP_KILL)) 
                return 1;

        return 0;
}
```

**四、capability权限隔离**

user namespace可以将主体进程的权限限定在一部分user ns范围之内，当做权限判断时，除了判断主体进程的cap\_effective的bit是否具备权限，还需要将user ns。以此达到capability权限隔离的目的。

判断主体是否操作客体的权限时，如在仅有一个uns（init\_user\_ns）的情况下，只需判断主体的cred\-\>cap\_effective中是否设置具体的权限掩码。如开启user namespace功能，在存在多个uns的场景，一切的判断都需要加上入参user\_namespace来限定所判断的环境。ns\_capable\(\)函数就是这个判断的核心函数。

因此作为客体资源，不论哪一种，都必须要有其所关联的user namespace，后续的权限判断也要基于此user namespace来判断

- Tasks （task\_struct\-\>real\_cred\-\>user\_ns）
- Files/inodes
- Sockets
- Message queues
- Shared memory segments
- Semaphores
- Keys
- 其他类型的 namespace\(UTS/PID/Mount/IPC/Network/Cgroup\)，都需要指定当前 ns 所属的 user namespace\(即 xxx\_ns\-\>user\_ns\)

ns\_capable判断当前进程是否早ns指定的用户命名空间中具备有效的参数cap指定的权利

```
bool ns_capable(struct user_namespace *ns, int cap)
{
        return ns_capable_common(ns, cap, true);
}

static bool ns_capable_common(struct user_namespace *ns, int cap, bool audit)
{
        int capable;

        capable = audit ? security_capable(current_cred(), ns, cap) :  //current_cred()表示当前进程的主体凭证，task->cred
                          security_capable_noaudit(current_cred(), ns, cap);
        if (capable == 0) {
                current->flags |= PF_SUPERPRIV;
                return true;
        }
        return false;
}

int security_capable(const struct cred *cred, struct user_namespace *ns,
                     int cap)
{
        return call_int_hook(capable, 0, cred, ns, cap, SECURITY_CAP_AUDIT); //实际是security_hook_heads.capable链表中注册的函数
}
//实际的校验函数是cap_capable()，该函数判断进程是否具备某一特定的cap
int cap_capable(const struct cred *cred, struct user_namespace *targ_ns,
                int cap, int audit)
{
        struct user_namespace *ns = targ_ns;
        for (;;) {
                //主体和客体是同一命名空间
                if (ns == cred->user_ns)
                        return cap_raised(cred->cap_effective, cap) ? 0 : -EPERM;

                //客体是最顶层的init_user_ns，直接不用判断返回无权限
                if (ns == &init_user_ns)
                        return -EPERM;
                /*
                 * The owner of the user namespace in the parent of the
                 * user namespace has all caps.
                 */
                //主体的uns是客体的uns父亲，并且主体进程的euid等于客体uns的owner
                if ((ns->parent == cred->user_ns) && uid_eq(ns->owner, cred->euid))
                        return 0;
                /*
                 * If you have a capability in a parent user ns, then you have
                 * it over all children user namespaces as well.
                 */
                ns = ns->parent;
        }

        /* We never get here */
}
```

根据cap\_capable函数可知，主体（进程）对客体（资源）的capability判断依据以下原则：

- 如果主体\(进程\)的 user ns 和客体的 user ns 是同一层级为兄弟关系\(即同一个父亲 user ns\)，或者主体的 user ns 比客体的 user ns 层级更多\(最顶层 level=0，逐级增加\)，则禁止操作。
- 如果主体\(进程\)的 user ns 和客体的 user ns 相同，则根据主体\(进程\)的 capability\( task\-\>cred\-\>cap\_effective \)是否有当前操作需要的能力来判断操作是否允许。
- 如果主体\(进程\)的 user ns 是客体的 user ns 的父节点，并且主体\(进程\)是客体的 user ns 的 owner，则客体对主体拥有所有的 capability 权限。

**五、参考资料：**

[https://lwn.net/Articles/532593/](https://lwn.net/Articles/532593/)

[https://www.kernel.org/doc/html/latest/security/credentials.html\#types\-of\-credentials](https://www.kernel.org/doc/html/latest/security/credentials.html#types-of-credentials)

[https://tinylab.org/user\-namespace/](https://tinylab.org/user-namespace/)
