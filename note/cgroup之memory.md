# cgroup之memory

基于linux5.8

```
struct cgroup_subsys memory_cgrp_subsys = {
        .css_alloc = mem_cgroup_css_alloc,
        .css_online = mem_cgroup_css_online,
        .css_offline = mem_cgroup_css_offline,
        .css_released = mem_cgroup_css_released,
        .css_free = mem_cgroup_css_free,
        .css_reset = mem_cgroup_css_reset,
        .can_attach = mem_cgroup_can_attach,
        .cancel_attach = mem_cgroup_cancel_attach,
        .post_attach = mem_cgroup_move_task,
        .bind = mem_cgroup_bind,
        .dfl_cftypes = memory_files,
        .legacy_cftypes = mem_cgroup_legacy_files,
        .early_init = 0,
};
```

```
struct page_counter {
        atomic_long_t usage;
        unsigned long min;
        unsigned long low;
        unsigned long high;
        unsigned long max;
        struct page_counter *parent;

        /* effective memory.min and memory.min usage tracking */
        unsigned long emin;
        atomic_long_t min_usage;
        atomic_long_t children_min_usage;

        /* effective memory.low and memory.low usage tracking */
        unsigned long elow;
        atomic_long_t low_usage;
        atomic_long_t children_low_usage;

        /* legacy */
        unsigned long watermark;
        unsigned long failcnt;
};

/*
* The memory controller data structure. The memory controller controls both
* page cache and RSS per cgroup. We would eventually like to provide
* statistics based on the statistics developed by Rik Van Riel for clock-pro,
* to help the administrator determine what knobs to tune.
*/
struct mem_cgroup {
        struct cgroup_subsys_state css;

        /* Private memcg ID. Used to ID objects that outlive the cgroup */
        struct mem_cgroup_id id;

        /* Accounted resources */
        struct page_counter memory;
        struct page_counter swap;

        /* Legacy consumer-oriented counters */
        struct page_counter memsw;
        struct page_counter kmem;
        struct page_counter tcpmem;

        /* Range enforcement for interrupt charges */
        struct work_struct high_work;

        unsigned long soft_limit;

        /* vmpressure notifications */
        struct vmpressure vmpressure;

        /*
         * Should the accounting and control be hierarchical, per subtree?
         */
        bool use_hierarchy;

        /*
         * Should the OOM killer kill all belonging tasks, had it kill one?
         */
        bool oom_group;

        /* protected by memcg_oom_lock */
        bool            oom_lock;
        int             under_oom;

        int     swappiness;
        /* OOM-Killer disable */
        int             oom_kill_disable;

        /* memory.events and memory.events.local */
        struct cgroup_file events_file;
        struct cgroup_file events_local_file;

        /* handle for "memory.swap.events" */
        struct cgroup_file swap_events_file;

        /* protect arrays of thresholds */
        struct mutex thresholds_lock;

        /* thresholds for memory usage. RCU-protected */
        struct mem_cgroup_thresholds thresholds;

        /* thresholds for mem+swap usage. RCU-protected */
        struct mem_cgroup_thresholds memsw_thresholds;

        /* For oom notifier event fd */
        struct list_head oom_notify;

        /*
         * Should we move charges of a task when a task is moved into this
         * mem_cgroup ? And what type of charges should we move ?
         */
        unsigned long move_charge_at_immigrate;
        /* taken only while moving_account > 0 */
        spinlock_t              move_lock;
        unsigned long           move_lock_flags;

        MEMCG_PADDING(_pad1_);

        /*
         * set > 0 if pages under this cgroup are moving to other cgroup.
         */
        atomic_t                moving_account;
        struct task_struct      *move_lock_task;

        /* Legacy local VM stats and events */
        struct memcg_vmstats_percpu __percpu *vmstats_local;

        /* Subtree VM stats and events (batched updates) */
        struct memcg_vmstats_percpu __percpu *vmstats_percpu;

        MEMCG_PADDING(_pad2_);

        atomic_long_t           vmstats[MEMCG_NR_STAT];
        atomic_long_t           vmevents[NR_VM_EVENT_ITEMS];
        /* memory.events */
        atomic_long_t           memory_events[MEMCG_NR_MEMORY_EVENTS];
        atomic_long_t           memory_events_local[MEMCG_NR_MEMORY_EVENTS];

        unsigned long           socket_pressure;

        /* Legacy tcp memory accounting */
        bool                    tcpmem_active;
        int                     tcpmem_pressure;

#ifdef CONFIG_MEMCG_KMEM
        /* Index in the kmem_cache->memcg_params.memcg_caches array */
        int kmemcg_id;
        enum memcg_kmem_state kmem_state;
        struct list_head kmem_caches;
#endif

#ifdef CONFIG_CGROUP_WRITEBACK
        struct list_head cgwb_list;
        struct wb_domain cgwb_domain;
        struct memcg_cgwb_frn cgwb_frn[MEMCG_CGWB_FRN_CNT];
#endif

        /* List of events which userspace want to receive */
        struct list_head event_list;
        spinlock_t event_list_lock;

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
        struct deferred_split deferred_split_queue;
#endif

        struct mem_cgroup_per_node *nodeinfo[0];
        /* WARNING: nodeinfo must be the last member here */
};
```

[https://www.cnblogs.com/lege/p/4329480.html](https://www.cnblogs.com/lege/p/4329480.html)

charge/uncharge

mem\_cgroup统计的对象主要是用户空间使用的内存，分匿名映射（anon page）和文件映射（page cache）两种类型的page。而这两种page又存在swap的情况。

至于其他的内存，则是由内核空间使用的，不在统计之列。
