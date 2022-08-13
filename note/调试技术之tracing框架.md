# 调试技术之tracing框架

**一、背景说明**

Linux trace中，最基础的就是：function tracer和trace event两种。鉴于他们搭建的良好的框架\(ringbuffer、tracefs\)，各种trace纷纷投奔而来。

tracer发展出了function、function\_graph、irqsoff、preemptoff、wakeup等一系列tracer。而event也发展出tracepoint、kprobe、uprobe等一系列的event。

**1.1 tracer**

tracer是利用编译时进行插桩的。在gcc使用了“\-pg”选项以后，会在每个函数的入口插入\_mcount\(\)函数调用，通过跳转指令的修改，可以做很多文章。

可以看到一个普通的系统，function tracer的插桩点为5万个左右，trace event的插桩点为1千个左右。function tracer的插桩点几乎大于trace event插桩点两个数量级，而且trace event的插桩点是每一个插桩点都是独立控制的，而function tracer的插桩点默认是一起控制的\(也可以通过set\_ftrace\_filter、set\_ftrace\_notrace来动态分开控制\)。

内核中有各种适用于不同场景的tracer：

```
root@lilong:/sys/kernel/debug/tracing# cat available_tracers
hwlat blk mmiotrace function_graph wakeup_dl wakeup_rt wakeup function nop
```

tracer的核心用于跟踪特定的行为特征，如函数调用、函数调用关系图、开关中断等。

**1.2 event**

trace event的插桩使用的是tracepoint机制，tracepoint是一种静态的插桩方法。他需要静态的定义桩函数，并且在插桩位置显式的调用。这种方法的好处是效率高、可靠，并且可以处于函数中的任何位置、方便的访问各种变量；坏处当然是不太灵活。kernel在重要节点的固定位置，插入了几百个trace event用于跟踪。基于此内核还引入了kprobe、uprobe等一系列的event。

event的核心在于跟踪函数内运行过程中具体数据的状态，打印变量。

**二、数据结构**

struct trace\_array代表了一组trace环境，其有独立的控制参数和输出buffer，管理着一套独立的tracer和event参数和入口，不同trace组的输出到各自的buffer，互不干扰，参数独立也不干扰。系统默认只创建了global\_trace，其关联的也是tracing顶层目录。用户可以在tracing/instance目录中通过创建目录的形式创建新的trace组并与所创建的目录关联，其下的文件可读写其隶属的struct trace\_array。

```
struct trace_array {
        struct list_head        list;
        char                    *name;
        struct trace_buffer     trace_buffer; //trace组管理的独立buffer，buffer_size_kb/buffer_total_size_kb与之关联
        unsigned long           max_latency;  //与文件tracing_max_latency关联
        arch_spinlock_t         max_lock;
        int                     buffer_disabled; 
        int                     stop_count;
        int                     clock_id; //与trace_clock文件关联
        int                     nr_topts;
        struct tracer           *current_trace; //指向当前的tracer
        unsigned int            trace_flags;
        unsigned char           trace_flags_index[TRACE_FLAGS_MAX_SIZE];
        unsigned int            flags;
        raw_spinlock_t          start_lock;
        struct dentry           *dir;
        struct dentry           *options;
        struct dentry           *percpu_dir;
        struct dentry           *event_dir;
        struct trace_options    *topts;
        struct list_head        systems;
        struct list_head        events;  //将所有的描述事件文件的结构struct trace_event_file通过链表链接起来。
                                //而trace_event_file->call指向具体的管理事件的数据结构struct trace_event_call。

        cpumask_var_t           tracing_cpumask; //与tracing_cpumask文件关联
        int                     ref;
#ifdef CONFIG_FUNCTION_TRACER
        struct ftrace_ops       *ops;
        struct trace_pid_list   __rcu *function_pids;
        /* function tracing enabled */
        int                     function_enabled;
#endif
};
```

tracer与trace\_array

```
struct tracer {
        const char              *name;
        int                     (*init)(struct trace_array *tr);
        void                    (*reset)(struct trace_array *tr);
        void                    (*start)(struct trace_array *tr);
        void                    (*stop)(struct trace_array *tr);
        int                     (*update_thresh)(struct trace_array *tr);
        void                    (*open)(struct trace_iterator *iter);
        void                    (*pipe_open)(struct trace_iterator *iter);
        void                    (*close)(struct trace_iterator *iter);
        void                    (*pipe_close)(struct trace_iterator *iter);
        ssize_t                 (*read)(struct trace_iterator *iter,
                                        struct file *filp, char __user *ubuf,
                                        size_t cnt, loff_t *ppos);
        ssize_t                 (*splice_read)(struct trace_iterator *iter,
                                               struct file *filp,
                                               loff_t *ppos,
                                               struct pipe_inode_info *pipe,
                                               size_t len,
                                               unsigned int flags);
        void                    (*print_header)(struct seq_file *m);
        enum print_line_t       (*print_line)(struct trace_iterator *iter);
        /* If you handled the flag setting, return 0 */
        int                     (*set_flag)(struct trace_array *tr,
                                            u32 old_flags, u32 bit, int set);
        /* Return 0 if OK with change, else return non-zero */
        int                     (*flag_changed)(struct trace_array *tr,
                                                u32 mask, int set);
        struct tracer           *next;
        struct tracer_flags     *flags;
        int                     enabled;
        int                     ref;
        bool                    print_max;
        bool                    allow_instances;
};
```

**三、框架分析**

**四、代码分析**

tracer的初始化

```
static __init int tracer_init_tracefs(void)
{
        struct dentry *d_tracer;

        //1、在debugfs中创建tracing目录，挂载后路径/sys/kernel/debug/tracing
        d_tracer = tracing_init_dentry(); 

        //2、在tracing目录中基于global_trace创建顶层的tracer和event框架文件
        init_tracer_tracefs(&global_trace, d_tracer);
        //该函数在tracing目录中围绕global_trace，创建顶层trace才有的文件，即instance中创建所没有的功能。
        //包括：（1）function tracer的动态跟踪部分功能 （2）function graph tracer （3）profile的部分
        ftrace_init_tracefs_toplevel(&global_trace, d_tracer);

        //3、在tracing目录中创建文件，并且只会在tracing顶层目录中创建一次
        trace_create_file("tracing_thresh", 0644, d_tracer,
                        &global_trace, &tracing_thresh_fops);
        trace_create_file("README", 0444, d_tracer,
                        NULL, &tracing_readme_fops);
        trace_create_file("saved_cmdlines", 0444, d_tracer,
                        NULL, &tracing_saved_cmdlines_fops);
        trace_create_file("saved_cmdlines_size", 0644, d_tracer,
                          NULL, &tracing_saved_cmdlines_size_fops);
        trace_enum_init();
        trace_create_enum_file(d_tracer);

#ifdef CONFIG_MODULES
        register_module_notifier(&trace_module_nb);
#endif
#ifdef CONFIG_DYNAMIC_FTRACE
        trace_create_file("dyn_ftrace_total_info", 0444, d_tracer,
                        &ftrace_update_tot_cnt, &tracing_dyn_info_fops);
#endif

        //4、在tracing中创建instances目录，instance目录的mkdir方法为：tracefs_dir_inode_operations->tracefs_ops.mkdir(instance_mkdir)
        //instance目录提供了一个可以创建独立的trace_buffers的入口，即在该目录中创建一个目录的同时(调用instance_mkdir)，
        //会创建一个struct trace_array数据结构，该结构类似于global_trace的作用但有所精简，伴随着struct trace_array相关的初始化，
        //会在该目录下创建一套与tracer和event相关的文件，但是使用独立于global_trace的buffer。
        create_trace_instances(d_tracer);

        //2、更新与global_trace关联的option参数
        update_tracer_options(&global_trace);

        return 0;
}
```

**五、参考资料：**

[https://www.cnblogs.com/hellokitty2/p/15624976.html](https://www.cnblogs.com/hellokitty2/p/15624976.html)

[https://blog.csdn.net/pwl999/article/details/80702365/](https://blog.csdn.net/pwl999/article/details/80702365/)
