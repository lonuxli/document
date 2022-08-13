# 进程调度之RT throttling

问题：出现sched: RT throttling activated， 23S后出现soft lockup

参考：[https://www.cnblogs.com/arnoldlu/p/9025981.html](https://www.cnblogs.com/arnoldlu/p/9025981.html)

```
struct task_struct {
...
    int prio, static_prio, normal_prio;
    unsigned int rt_priority;
    const struct sched_class *sched_class;
    struct sched_entity se;
    struct sched_rt_entity rt;
#ifdef CONFIG_CGROUP_SCHED
    struct task_group *sched_task_group;
#endif
    struct sched_dl_entity dl;
...
}
```

```
struct rq {
        /* runqueue lock: */
        raw_spinlock_t lock;

        /*
         * nr_running and cpu_load should be in the same cacheline because
         * remote CPUs use both these fields when doing load calculation.
         */
        unsigned int nr_running;
#ifdef CONFIG_NUMA_BALANCING
        unsigned int nr_numa_running;
        unsigned int nr_preferred_running;
#endif
        #define CPU_LOAD_IDX_MAX 5
        unsigned long cpu_load[CPU_LOAD_IDX_MAX];
#ifdef CONFIG_NO_HZ_COMMON
#ifdef CONFIG_SMP
        unsigned long last_load_update_tick;
#endif /* CONFIG_SMP */
        unsigned long nohz_flags;
#endif /* CONFIG_NO_HZ_COMMON */
#ifdef CONFIG_NO_HZ_FULL
        unsigned long last_sched_tick;
#endif
        /* capture load from *all* tasks on this cpu: */
        struct load_weight load;
        unsigned long nr_load_updates;
        u64 nr_switches;

        struct cfs_rq cfs;
        struct rt_rq rt;    //rt调度队列
        struct dl_rq dl;

#ifdef CONFIG_FAIR_GROUP_SCHED
        /* list of leaf cfs_rq on this cpu: */
        struct list_head leaf_cfs_rq_list;
#endif /* CONFIG_FAIR_GROUP_SCHED */
        unsigned long nr_uninterruptible;

        struct task_struct *curr, *idle, *stop;
        unsigned long next_balance;
        struct mm_struct *prev_mm;

        unsigned int clock_skip_update;
        u64 clock;    //cpu时间
        u64 clock_task;    //

        atomic_t nr_iowait;

#ifdef CONFIG_SMP
        struct root_domain *rd;
        struct sched_domain *sd;

        unsigned long cpu_capacity;
        unsigned long cpu_capacity_orig;

        struct callback_head *balance_callback;

        unsigned char idle_balance;
        /* For active balancing */
        int active_balance;
        int push_cpu;
        struct cpu_stop_work active_balance_work;
        /* cpu of this runqueue: */
        int cpu;
        int online;

        struct list_head cfs_tasks;

        u64 rt_avg;
        u64 age_stamp;
        u64 idle_stamp;
        u64 avg_idle;
       /* This is used to determine avg_idle's max value */
        u64 max_idle_balance_cost;
#endif

#ifdef CONFIG_IRQ_TIME_ACCOUNTING
        u64 prev_irq_time;
#endif
#ifdef CONFIG_PARAVIRT
        u64 prev_steal_time;
#endif
#ifdef CONFIG_PARAVIRT_TIME_ACCOUNTING
        u64 prev_steal_time_rq;
#endif

        /* calc_load related fields */
        unsigned long calc_load_update;
        long calc_load_active;

#ifdef CONFIG_SCHED_HRTICK
#ifdef CONFIG_SMP
        int hrtick_csd_pending;
        struct call_single_data hrtick_csd;
#endif
        struct hrtimer hrtick_timer;
#endif

#ifdef CONFIG_SCHEDSTATS
        /* latency stats */
        struct sched_info rq_sched_info;
        unsigned long long rq_cpu_time;
        /* could above be rq->cfs_rq.exec_clock + rq->rt_rq.rt_runtime ? */

        /* sys_sched_yield() stats */
        unsigned int yld_count;

        /* schedule() stats */
        unsigned int sched_count;
        unsigned int sched_count;
        unsigned int sched_goidle;

        /* try_to_wake_up() stats */
        unsigned int ttwu_count;
        unsigned int ttwu_local;
#endif

#ifdef CONFIG_SMP
        struct llist_head wake_list;
#endif

#ifdef CONFIG_CPU_IDLE
        /* Must be inspected within a rcu lock section */
        struct cpuidle_state *idle_state;
#endif
};
```

```
/* Real-Time classes' related field in a runqueue: */
struct rt_rq {
        struct rt_prio_array active;
        unsigned int rt_nr_running;
        unsigned int rr_nr_running;
#if defined CONFIG_SMP || defined CONFIG_RT_GROUP_SCHED
        struct {
                int curr; /* highest queued rt task prio */
#ifdef CONFIG_SMP
                int next; /* next highest */
#endif
        } highest_prio;
#endif
#ifdef CONFIG_SMP
        unsigned long rt_nr_migratory;
        unsigned long rt_nr_total;
        int overloaded;
        struct plist_head pushable_tasks;
#endif /* CONFIG_SMP */
        int rt_queued;

        int rt_throttled;
        u64 rt_time;
        u64 rt_runtime;
        /* Nests inside the rq lock: */
        raw_spinlock_t rt_runtime_lock;

#ifdef CONFIG_RT_GROUP_SCHED
        unsigned long rt_nr_boosted;

        struct rq *rq;
        struct task_group *tg;
#endif
};
```

```
struct rt_bandwidth {
        /* nests inside the rq lock: */
        raw_spinlock_t          rt_runtime_lock;
        ktime_t                 rt_period;
        u64                     rt_runtime;
        struct hrtimer          rt_period_timer;
        unsigned int            rt_period_active;
};                 
```

```
/* task group related information */
struct task_group {
        struct cgroup_subsys_state css;

#ifdef CONFIG_FAIR_GROUP_SCHED
        /* schedulable entities of this group on each cpu */
        struct sched_entity **se;
        /* runqueue "owned" by this group on each cpu */
        struct cfs_rq **cfs_rq;
        unsigned long shares;

#ifdef  CONFIG_SMP
        /*
         * load_avg can be heavily contended at clock tick time, so put
         * it in its own cacheline separated from the fields above which
         * will also be accessed at each tick.
         */
        atomic_long_t load_avg ____cacheline_aligned;
#endif
#endif

#ifdef CONFIG_RT_GROUP_SCHED
        struct sched_rt_entity **rt_se;
        struct rt_rq **rt_rq;

        struct rt_bandwidth rt_bandwidth;
#endif

        struct rcu_head rcu;
        struct list_head list;

        struct task_group *parent;
        struct list_head siblings;
        struct list_head children;

#ifdef CONFIG_SCHED_AUTOGROUP
        struct autogroup *autogroup;
#endif

        struct cfs_bandwidth cfs_bandwidth;
};
```

```
struct sched_rt_entity {
        struct list_head run_list;
        unsigned long timeout;    --watchdog计数，用于判断当前进程时间是否超过RLIMIT_RTTIME
        unsigned long watchdog_stamp;
        unsigned int time_slice;    --针对RR调度策略的调度时隙 时间到了就冲将当前任务放到该优先级队尾，选择下一个任务进行运行
        unsigned short on_rq;
        unsigned short on_list;
        
        struct sched_rt_entity *back; --dequeue_rt_stack()中作为临时变量使用
#ifdef CONFIG_RT_GROUP_SCHED
        struct sched_rt_entity  *parent; --指向上一层调度实体
        /* rq on which this entity is (to be) queued: */
        struct rt_rq            *rt_rq;    --当前实时调度实体所在的就绪队列
        /* rq "owned" by this entity/group: */
        struct rt_rq            *my_q;    --当前实时调度实体的子调度实体所在的就绪队列
#endif  
};
```

```
  99 void update_rq_clock(struct rq *rq)
100 {
101 ▼       s64 delta;
102
105 ▼       if (rq->clock_skip_update & RQCF_ACT_SKIP)
106 ▼       ▼       return;
107
108 ▼       delta = sched_clock_cpu(cpu_of(rq)) - rq->clock;  //sched_clock_cpu这个时间应该是cpu运行时间戳
109 ▼       if (delta < 0)
110 ▼       ▼       return;
111 ▼       rq->clock += delta;
112 ▼       update_rq_clock_task(rq, delta);
113 }

787 static void update_rq_clock_task(struct rq *rq, s64 delta)
788 {
#ifdef CONFIG_IRQ_TIME_ACCOUNTING
794 ▼       s64 steal = 0, irq_delta = 0;
797 ▼       irq_delta = irq_time_read(cpu_of(rq)) - rq->prev_irq_time; //irq_time_read（cpu）：cpu中断上下文运行总共时间 - 
798
814 ▼       if (irq_delta > delta)
815 ▼       ▼       irq_delta = delta;
816
817 ▼       rq->prev_irq_time += irq_delta;
818 ▼       delta -= irq_delta;
#endif
833 ▼       rq->clock_task += delta;  //如果系统可以统计中断上下文时间，则rq->clock_task中不包括中断上下问题时间，否则包括
#ifdef CONFIG_IRQ_TIME_ACCOUNTING
836 ▼       if ((irq_delta + steal) && sched_feat(NONTASK_CAPACITY))
837 ▼       ▼       sched_rt_avg_update(rq, irq_delta + steal);
#endif
839 }
```

\_\_schedule

```
3366 ▼       switch_count = &prev->nivcsw;
3367 ▼       if (!preempt && prev->state) {
3368 ▼       ▼       if (unlikely(signal_pending_state(prev->state, prev))) {
3369 ▼       ▼       ▼       prev->state = TASK_RUNNING;
3370 ▼       ▼       } else {
3371 ▼       ▼       ▼       deactivate_task(rq, prev, DEQUEUE_SLEEP);  如果是主动调度出去，且当前状态位非running状态，将task从当前队列中移除
3372 ▼       ▼       ▼       prev->on_rq = 0;
3373
3374 ▼       ▼       ▼       /*
3375 ▼       ▼       ▼        * If a worker went to sleep, notify and ask workqueue
3376 ▼       ▼       ▼        * whether it wants to wake up a task to maintain
3377 ▼       ▼       ▼        * concurrency.
3378 ▼       ▼       ▼        */
3379 ▼       ▼       ▼       if (prev->flags & PF_WQ_WORKER) {
3380 ▼       ▼       ▼       ▼       struct task_struct *to_wakeup;
3381
3382 ▼       ▼       ▼       ▼       to_wakeup = wq_worker_sleeping(prev);
3383 ▼       ▼       ▼       ▼       if (to_wakeup)
3384 ▼       ▼       ▼       ▼       ▼       try_to_wake_up_local(to_wakeup, cookie);
3385 ▼       ▼       ▼       }
3386 ▼       ▼       }
3387 ▼       ▼       switch_count = &prev->nvcsw;
3388 ▼       }
```
