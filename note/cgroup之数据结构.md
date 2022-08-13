# cgroup之数据结构

基于Linux5.8版本总结

**一、基本概念**

1.任务（task）。在cgroups中，任务就是系统的一个进程。

2.控制族群（control group）。控制族群就是一组按照某种标准划分的进程。Cgroups中的资源控制都是以控制族群为单位实现。一个进程可以加入到某个控制族群，也从一个进程组迁移到另一个控制族群。一个进程组的进程可以使用cgroups以控制族群为单位分配的资源，同时受到cgroups以控制族群为单位设定的限制。

3.层级（hierarchy）。控制族群可以组织成hierarchical的形式，既一颗控制族群树（即一颗cgroup树）。控制族群树上的子节点控制族群是父节点控制族群的孩子，继承父控制族群的特定的属性。

4.子系统（subsytem）。一个子系统就是一个资源控制器，比如cpu子系统就是控制cpu时间分配的一个控制器。子系统必须附加（attach）到一个层级上才能起作用，一个子系统附加到某个层级以后，这个层级上的所有控制族群都受到这个子系统的控制。

相互关系

1.每次在系统中创建新层级时，该系统中的所有任务都是那个层级的默认 cgroup（我们称之为 root cgroup ，此cgroup在创建层级时自动创建，后面在该层级中创建的cgroup都是此cgroup的后代）的初始成员。

2.一个子系统最多只能附加到一个层级。所以层级的数量是小于或等于子系统类型的数量的。

3.一个层级可以附加多个子系统。这样的好处是减少层级的数量，实际使用过程中可能并不需要那么多独立子系统构成的层级。如下cpu和cpuacct可以附加到同一层级中。

4.一个任务可以是多个cgroup的成员，但是这些cgroup必须在不同的层级。层级中cgroup间限制的是相同的资源只是具体限制值不同，不能受多个cgroup限制而出现冲突。

5.系统中的进程（任务）创建子进程（任务）时，该子任务自动成为其父进程所在 cgroup 的成员。然后可根据需要将该子任务移动到不同的 cgroup 中，但开始时它总是继承其父任务的cgroup。

![64cae5348821ee40c2136808bde6653b.png](image/64cae5348821ee40c2136808bde6653b.png)

如下图，task1和task2两个任务都在在cpu cgroup4和memory cgroup5中，task group应该是虚拟出来的，实际并没有任务组。

                              ![cgroup_hierarchy.png](image/cgroup_hierarchy.png)

**二、数据结构**

```
struct task_struct {
#ifdef CONFIG_CGROUPS
        /* Control Group info protected by css_set_lock: */
        struct css_set __rcu            *cgroups;  //指向task对应的css_set
        /* cg_list protected by css_set_lock and tsk->alloc_lock: */
        struct list_head                cg_list; //加入css_set的链表，使用相同css_set的task串联起来
#endif
}

struct css_set {
        struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];  //指向css_set关联的子系统，只有与之有关的子系统项指针才有效

        /* reference count */
        refcount_t refcount;
        /* internal task count, protected by css_set_lock */
        int nr_tasks;  //使用该css_set的总的task数
        struct list_head tasks; //串联用该css_set的task
        ...
}

struct cgroup_subsys_state {
        /* PI: the cgroup that this css is attached to */
        struct cgroup *cgroup;  //子系统关联的cgroup数据结构

        /* PI: the cgroup subsystem that this css is attached to */
        struct cgroup_subsys *ss; //子系统关联具体的子系统方法

        /* reference count - access via css_[try]get() and css_put() */
        struct percpu_ref refcnt;

        /* siblings list anchored at the parent's ->children */
        struct list_head sibling;  //用于建立该子系统的树形关系图
        struct list_head children;
        struct cgroup_subsys_state *parent;
};

struct cgrp_cset_link {
        /* the cgroup and css_set this link associates */
        struct cgroup           *cgrp; //指向该cgrp_cset_link关联的cgroup数据结构
        struct css_set          *cset;  //指向该cgrp_cset_link关联的css_set数据结构

        /* list of cgrp_cset_links anchored at cgrp->cset_links */
        struct list_head        cset_link; //串联起同属于同一个css_set的cgrp_cset_link类型数据结构

        /* list of cgrp_cset_links anchored at css_set->cgrp_links */
        struct list_head        cgrp_link;  //串联起同属于同一个cgroup的cgrp_cset_link类型数据结构
};

struct cgroup_root {
        struct kernfs_root *kf_root;

        /* The bitmask of subsystems attached to this hierarchy */
        unsigned int subsys_mask;

        /* Unique id for this hierarchy. */
        int hierarchy_id;

        /* The root cgroup.  Root is destroyed on its release. */
        struct cgroup cgrp;

        /* for cgrp->ancestor_ids[0] */
        u64 cgrp_ancestor_id_storage;

        /* Number of cgroups in the hierarchy, used only for /proc/cgroups */
        atomic_t nr_cgrps;

        /* A list running through the active hierarchies */
        struct list_head root_list;

        /* Hierarchy-specific flags */
        unsigned int flags;

        /* The path to use for release notifications. */
        char release_agent_path[PATH_MAX];

        /* The name for this hierarchy - may be empty */
        char name[MAX_CGROUP_ROOT_NAMELEN];  //该hierarchy
};

struct cgroup_subsys {
        struct cgroup_subsys_state *(*css_alloc)(struct cgroup_subsys_state *parent_css);
        int (*css_online)(struct cgroup_subsys_state *css);
        void (*css_offline)(struct cgroup_subsys_state *css);
        void (*css_released)(struct cgroup_subsys_state *css);
        void (*css_free)(struct cgroup_subsys_state *css);
        void (*css_reset)(struct cgroup_subsys_state *css);
        void (*css_rstat_flush)(struct cgroup_subsys_state *css, int cpu);
        int (*css_extra_stat_show)(struct seq_file *seq,
                                   struct cgroup_subsys_state *css);

        int (*can_attach)(struct cgroup_taskset *tset);
        void (*cancel_attach)(struct cgroup_taskset *tset);
        void (*attach)(struct cgroup_taskset *tset);
        void (*post_attach)(void);
        int (*can_fork)(struct task_struct *task,
                        struct css_set *cset);
        void (*cancel_fork)(struct task_struct *task, struct css_set *cset);
        void (*fork)(struct task_struct *task);
        void (*exit)(struct task_struct *task);
        void (*release)(struct task_struct *task);
        void (*bind)(struct cgroup_subsys_state *root_css);
        ...
}
```

**三、数据结构关系**

![cgroup_arch2.jpg](image/cgroup_arch2.jpg)

**四、整体框架�**�

 

                            

![1ebb1c6450022f58107851b5b13ca2f7.png](image/1ebb1c6450022f58107851b5b13ca2f7.png)

1、两个进程组，组A包括1、2、3进程;组B包括4、5、6进程

2、系统中包括一个mem\_cpu层级（包括memory\+cpu两个子系统）和其他层级，该层级有1\-6个cgroup

3、组A的进程和组B均加入mem\_cpu层级的cgroup5，以及其他层级;cgroup5中的cset\_link链表串联两个与之有关的cgrup\_set\_link，同样css\_set也通过cgrp\_link串联多个cgrup\_set\_link；每对存在关联的cgroup与css\_set之间都会建立一个cgrup\_set\_link来将其关联起来。

4、mem\_cpu层级中，每个cgroup有两个与之对应的cgroup\_subsys\_state，分别为memory和cpu，每个cgroup\_subsys\_state代表cgroup系统中一个子系统示例。在子系统如memory中通过parent/children/slibing指针串联，形成树形关系。cgroup\_subsys\_state通过cgroup元素指向其cgroup。

5、系统中每个cgroup子系统都有一个唯一的cgroup\_subsys，维护具体子系统相关的数据及方法，这些cgroup\_subsys组成一个全局数组。每个cgroup\_subsys\_state实例都会指向其对应的子系统cgroup\_subsys。
