# 调试技术之fracerc桩点及函数转移路径

**一、背景**

ftrace/kprobe等等跟踪工具的原理无非是进行函数插桩，本文将梳理ftrace framework的桩点变化、管理及其函数跳转路径。

![ftrace_arch.png](image/ftrace_arch.png)

**二、trace 插桩**

**2.1 编译时插桩**

使用gcc的\-pg选项让内核中的每个函数\(除部分notrace修饰的函数外\)在执行前都调用一个特殊的插桩函数\_mcount\(\)。

![mcount.png](image/mcount.png)

如果每个函数都调用这个函数，势必会影响性能，所以内核支持动态开启函数跟踪。配置选项为CONFIG\_DYNAMIC\_FTRACE。在编译期间，使用了一个脚本recordmcount.pl将所有的mcount调用位置点记录在一个表里面。在内核启动阶段会把mcount的替换为NOP指令。当用户通过proc开启函数跟踪功能后，再把NOP替换为对应函数调用。

**2.2 桩点地址规整**

具体\_\_gnu\_mcount\_nc桩点函数的地址最终会保存在在\_\_start\_mcount\_loc和\_\_stop\_mcount\_loc之间，注意保存的是被插桩函数的首地址\+桩点相对函数首地址的偏移，保存数据的位置属于内核的.init.data段，在初始化完成之后会被释放掉。初始化收集过程如下：

1、把对应的.o文件的mcount记录到一个\_\_mcount\_loc seciton中。

2、把\_\_mcount\_loc seciton编译成.o文件链接到源object文件中。

3、把所有的object中的\_\_mcount\_loc section链接到vmlinx中的\_\_start\_mcount\_loc和 \_\_stop\_mcount\_loc区间内，存放在.init.data段中。

4、把vmlinux的符号替换为实际的地址。

![093c75e4ce35ab42ca7281a04f4a779a.png](image/093c75e4ce35ab42ca7281a04f4a779a.png)

![615d8bdf2171f6869331d0274fb9f2e2.png](image/615d8bdf2171f6869331d0274fb9f2e2.png)

**2.3 桩点的管理**

内核函数众多，如ubuntu 18.04发行版的4.15版本内核的大致为4.7W个左右的函数存在桩点，为了更好的管理桩点，如知道桩点是否需要被跟踪，具体被跟踪的方式等等，可能需要一些额外的内存来标记这些信息，\_start\_mcount\_loc和\_\_stop\_mcount\_loc之间的这点空间是远远不够的，因此内核每个插桩点都有一个struct dyn\_ftrace来管理，内核在初始化的时候会根据插桩点的个数来分配合适的物理page，用这些物理page来存放struct dyn\_ftrace结构体，同时会把\_start\_mcount\_loc和\_\_stop\_mcount\_loc之间的ip拷贝到struct dyn\_ftrace结构体中的ip中，之后被struct dyn\_ftrace接管。同时会把\_\_start\_mcount\_loc和\_\_stop\_mcount\_loc之间的物理内存释放掉。

```
struct dyn_ftrace {
        unsigned long           ip; /* address of mcount call-site */
        unsigned long           flags;
        struct dyn_arch_ftrace  arch;  //大部分为空
};
```

**ip**:插桩点对应的虚拟地址。

**flags**:分为两部分，低bit部分表示引用计数。如果有ftrace\_ops会操作到该ip，引用计数会加1。如果ref\_cnt大于0，插桩点就需要使能了。高bit部分表示不同的flag。

strcut dyn\_ftrace的出现方便了桩点信息的管理，但是dyn\_ftrace本身也需要妥善管理，一方面是需要连续考虑存储的内存空间，解决办法是专门申请物理页面来存储dyn\_ftrace，第二方面需要根据ip地址信息快速查找dyn\_ftrace数据结构，那么给ip做个排序可以方便查找。管理dyn\_ftrace和物理页面的数据结构为struct ftrace\_page。

```
struct ftrace_page {
        struct ftrace_page      *next;
        struct dyn_ftrace       *records;
        int                     index;
        int                     size;
};
```

**next:**指向下一个strct ftrace\_page结构体，所有的插桩点是由多个struct ftrace\_page共同管理的。

**records**:管理插桩点物理page对应的起始地址，即alloc\_page的返回值。records指向的是2^order \(order从0开始为\)大小的物理页面。

**index:**所管理实际插桩点的个数。

**size:**所管理的最大的插桩点的个数，最大插桩点个数为records指向的物理页面大小决定的。

![6fcc505556fd4e10b55d26d978deb569.png](image/6fcc505556fd4e10b55d26d978deb569.png)

桩点的整体管理模型如上图，数据的初始化主要有两个时机，一是内核启动时调用ftrace\_init\(\)初始所有内核镜像中的桩点，二是驱动插入时调用ftrace\_module\_init初始化驱动中的桩点。

```
void __init ftrace_init(void)
{
        count = __stop_mcount_loc - __start_mcount_loc;
        pr_info("ftrace: allocating %ld entries in %ld pages\n",
                count, count / ENTRIES_PER_PAGE + 1);
        ret = ftrace_process_locs(NULL,
                                  __start_mcount_loc,
                                  __stop_mcount_loc);
}

void ftrace_module_init(struct module *mod)
{
        ftrace_process_locs(mod, mod->ftrace_callsites,
                            mod->ftrace_callsites + mod->num_ftrace_callsites);
}
```

通过ftrace\_process\_locs初试化start\-\>end地址见存的桩点数据。

```
static int ftrace_process_locs(struct module *mod,
                               unsigned long *start,
                               unsigned long *end)
{
        count = end - start;
        //对桩点按地址进行排序，方便后续搜索，模块的桩点地址比内核镜像的小？所以两者之间不用排了？
        sort(start, count, sizeof(*start), ftrace_cmp_ips, NULL);  

        //根据桩点个数，分配ftrace_page管理数据结构和物理页面，并返回头部的ftrace_page
        start_pg = ftrace_allocate_pages(count); 
        mutex_lock(&ftrace_lock);

        if (!mod) {
                //如果当前是内核启动时镜像，初始化全局指针ftrace_pages/ftrace_pages_start
                //ftrace_pages指向链表尾
                //ftrace_pages_start指向链表头
                ftrace_pages = ftrace_pages_start = start_pg;
        } else {
                //如果是模块插入时，将模块的strct ftrace_page挂在链表尾（ftrace_pages）后面
                //那如果是多个模块反复插拔乱序，ftrace_page链表中的地址是否会排序是否会乱掉？
                ftrace_pages->next = start_pg;
        }

        p = start;
        pg = start_pg;
        while (p < end) {
                addr = ftrace_call_adjust(*p++); //取桩点地址

                if (pg->index == pg->size) {
                        //在前面ftrace_page分配时，pg->size中已经存着该ftrace_page的容量
                        pg = pg->next;
                }

                //利用当前遍历的机会，同时计算出当前ftrace_page中有效的dyn_trace的数量放在pg->index中
                rec = &pg->records[pg->index++];  
                rec->ip = addr;  
        }

        ftrace_pages = pg;  //更新ftrace_pages为链表尾
        ftrace_update_code(mod, start_pg); //==》将所有的桩点置为nop，关闭对一级桩跳转分支的调用
        mutex_unlock(&ftrace_lock);
}
```

通过ftrace\_allocate\_pages\(\)为num\_to\_init个dyn\_trace分配物理页面和管理数据结构。

```
static struct ftrace_page * ftrace_allocate_pages(unsigned long num_to_init)
{
        struct ftrace_page *start_pg;
        struct ftrace_page *pg;
        int order;
        int cnt;

        //循环为num_to_init个dyn_trace分配物理内存，可能需要多个ftrace_page结构才能放的下，将这些ftrace_page用链表串联
        start_pg = pg = kzalloc(sizeof(*pg), GFP_KERNEL);
        for (;;) {
                //尝试为num_to_init个dyn_trace分配物理内存，但实际考虑内存浪费和申请难度，可能申请到的小于num_to_init
                cnt = ftrace_allocate_records(pg, num_to_init);
                if (cnt < 0)
                        goto free_pages;

                num_to_init -= cnt;  
                if (!num_to_init)
                        break;

                pg->next = kzalloc(sizeof(*pg), GFP_KERNEL);
                if (!pg->next)
                        goto free_pages;
                pg = pg->next;  //加入链表
        }

        return start_pg;
}

static int ftrace_allocate_records(struct ftrace_page *pg, int count)
{
        int order;
        int cnt;

        //先根据struct dyn_trace的count数，计算出需要物理页面order
        order = get_count_order(DIV_ROUND_UP(count, ENTRIES_PER_PAGE));

        //如果申请的物理页面数量过多，最后有物理页面并未实际使用到，则降低物理页面的order分配更小的，防止浪费物理内存
        while ((PAGE_SIZE << order) / ENTRY_SIZE >= count + ENTRIES_PER_PAGE)
                order--;
again:
        pg->records = (void *)__get_free_pages(GFP_KERNEL | __GFP_ZERO, order);

        //如果连续的大的物理页面分配失败，则尝试减小页面粒度，重试分配
        if (!pg->records) {
                if (!order)
                        return -ENOMEM;
                order >>= 1;
                goto again;
        }

        cnt = (PAGE_SIZE << order) / ENTRY_SIZE;
        pg->size = cnt;

        if (cnt > count)
                cnt = count;  //将返回值cnt更新为实际的数量

        return cnt;
}
```

**2.4 tracer设置并初始化**

对于用户，要想使用tracer功能，首先是改变当前的tracer的指向，如"echo function \>  current\_tracer "通过文件设置当前为function tracer，实际是调用tracing\_set\_tracer\(\)函数进行设置。设置的过程中首先是复位原有的tracer，其次调用tracer\-\>init对当前新加载的tracer进行初始化，以function tracer为例，其调用function\_trace\_init\(\)进行初始化。

```
static int function_trace_init(struct trace_array *tr)
{
        ftrace_func_t func;
        //func为保存trace信息到ringbuffer中的回调函数
        if (tr->flags & TRACE_ARRAY_FL_GLOBAL &&
            func_flags.val & TRACE_FUNC_OPT_STACK)
                func = function_stack_trace_call; 
        else
                func = function_trace_call;

        ftrace_init_array_ops(tr, func); //主要是tr->ops->func = func;

        tr->trace_buffer.cpu = get_cpu();
        put_cpu();

        tracing_start_cmdline_record();
        tracing_start_function_trace(tr);
        return 0;
}

static void tracing_start_function_trace(struct trace_array *tr)
{
        tr->function_enabled = 0;
        register_ftrace_function(tr->ops);  //
        tr->function_enabled = 1;
}

int register_ftrace_function(struct ftrace_ops *ops)
{
        ftrace_ops_init(ops);
        mutex_lock(&ftrace_lock);
        ret = ftrace_startup(ops, 0);
        mutex_unlock(&ftrace_lock);
        return ret;
}

static int ftrace_startup(struct ftrace_ops *ops, int command)
{
        int ret;
         __register_ftrace_function(ops); //主要目的是注册ops

        ftrace_start_up++;

        ops->flags |= FTRACE_OPS_FL_ENABLED | FTRACE_OPS_FL_ADDING;

        ret = ftrace_hash_ipmodify_enable(ops);
        if (ret < 0) {
                /* Rollback registration process */
                __unregister_ftrace_function(ops);
                ftrace_start_up--;
                ops->flags &= ~FTRACE_OPS_FL_ENABLED;
                return ret;
        }
        if (ftrace_hash_rec_enable(ops, 1))
                command |= FTRACE_UPDATE_CALLS;
        ftrace_startup_enable(command);  //根据command中的标志更新一级/二级桩跳转点的跳转指令（函数）
        ops->flags &= ~FTRACE_OPS_FL_ADDING;
        return 0;
}

static int __register_ftrace_function(struct ftrace_ops *ops)
{
#ifndef CONFIG_DYNAMIC_FTRACE_WITH_REGS
        if (ops->flags & FTRACE_OPS_FL_SAVE_REGS &&
            !(ops->flags & FTRACE_OPS_FL_SAVE_REGS_IF_SUPPORTED))
                return -EINVAL;
        if (ops->flags & FTRACE_OPS_FL_SAVE_REGS_IF_SUPPORTED)
                ops->flags |= FTRACE_OPS_FL_SAVE_REGS;
#endif

        if (!core_kernel_data((unsigned long)ops))
                ops->flags |= FTRACE_OPS_FL_DYNAMIC;

        if (ops->flags & FTRACE_OPS_FL_PER_CPU) {
                if (per_cpu_ops_alloc(ops))
                        return -ENOMEM;
        }
        //将ops加入到全局的ftrace_ops_list链表中
        add_ftrace_ops(&ftrace_ops_list, ops);

        //将原始的ops->func保存至ops->saved_func，如果用户设置了通过pid筛选trace输出
        //则将ops->func改为ftrace_pid_func，ftrace_pid_func本质上只是将原始的ops->func（即ops->saved_func）
        //包裹了下，根据pid进行调用ops->saved_func。
        ops->saved_func = ops->func;
        if (ftrace_pids_enabled(ops))
                ops->func = ftrace_pid_func;

        ftrace_update_trampoline(ops);

        if (ftrace_enabled) //判断/proc/sys/kernel/ftrace_enabled配置是否使能
                update_ftrace_function(); //根据实际情况更新二级桩分支跳转函数

        return 0;
}
```

ftrace\_startup\_enable对所有的一级/二级桩点指令可能会进行修改，一旦修改，在ring\_bufer开启的情况下，则会真正往ring\_buffer中写入trace信息。

```
static void ftrace_startup_enable(int command)
{
        //ftrace_trace_function保存着更新后的二级桩跳转分支函数，saved_ftrace_func保存着之前的二级桩跳转分支函数
        //如果两者不一样，则需要附加上FTRACE_UPDATE_TRACE_FUNC标志，以让后续能按照正确的顺序修改指令。
        if (saved_ftrace_func != ftrace_trace_function) {
                saved_ftrace_func = ftrace_trace_function;
                command |= FTRACE_UPDATE_TRACE_FUNC;
        }
        ftrace_run_update_code(command);
}

static void ftrace_run_update_code(int command)
{
        //代码段指令修改前的准备工作，如有些平台会将代码段置为可写
        ftrace_arch_code_modify_prepare();

        arch_ftrace_update_code(command);

        //代码段指令修改后的收尾工作，如有些平台会将代码段恢复只读
        ftrace_arch_code_modify_post_process();
}

void arch_ftrace_update_code(int command)
{  
        //由于是假定是从function tracer调下来的，所以只可能会有
        //FTRACE_UPDATE_CALLS，FTRACE_UPDATE_TRACE_FUNC两个标志            
        ftrace_modify_all_code(command);
}     
```

![4e470bd41d75dee36daa8f40dc2fc189.png](image/4e470bd41d75dee36daa8f40dc2fc189.png)

**2.5 桩点指令修改策略**

桩点具体的更新由ftrace\_modify\_all\_code\(\)函数来完成，其在整个ftrace指令跳转框架中的位置如上图右边，参数command中的标志决定了桩点具体的指令情况。

```
enum {
        FTRACE_UPDATE_CALLS             = (1 << 0),   //遍历所有函数桩点，使能桩点函数调用
        FTRACE_DISABLE_CALLS            = (1 << 1),   //遍历所有函数桩点，失能桩点函数调用
        FTRACE_UPDATE_TRACE_FUNC        = (1 << 2),   //function trace位置（地址&ftrace_call）处理指令有修改
        //下面两条指令分别为function graph trace的使能和失能
        FTRACE_START_FUNC_RET           = (1 << 3),   //function graph trace位置（地址&ftrace_graph_call）处理设置为"bl ftrace_graph_caller"
        FTRACE_STOP_FUNC_RET            = (1 << 4),   //function graph trace位置（地址&ftrace_graph_call）处理设置为nop
};

void ftrace_modify_all_code(int command)
{
        int update = command & FTRACE_UPDATE_TRACE_FUNC;
        int err = 0;

                //如果function trace处理指令有修改，先将指令设置为"bl ftrace_ops_list_func"
        //ftrace_ops_list_func函数会检查hash链表确保。。。。？
        if (update) {  
                
                err = ftrace_update_ftrace_func(ftrace_ops_list_func);
                if (FTRACE_WARN_ON(err))
                        return;
        }

        if (command & FTRACE_UPDATE_CALLS)
                ftrace_replace_code(1);
        else if (command & FTRACE_DISABLE_CALLS)
                ftrace_replace_code(0);

        //全局变量ftrace_trace_function保存了function trace位置（地址&ftrace_call）处将要设置的跳转函数
        if (update && ftrace_trace_function != ftrace_ops_list_func) {
                function_trace_op = set_function_trace_op;
                smp_wmb();
                /* If irqs are disabled, we are in stop machine */
                if (!irqs_disabled())
                        smp_call_function(ftrace_sync_ipi, NULL, 1);
                err = ftrace_update_ftrace_func(ftrace_trace_function);  //设置&ftrace_call处的指令值
                if (FTRACE_WARN_ON(err))
                        return;
        }

        if (command & FTRACE_START_FUNC_RET)
                err = ftrace_enable_ftrace_graph_caller(); //function graph trace的跳转使能
        else if (command & FTRACE_STOP_FUNC_RET)
                err = ftrace_disable_ftrace_graph_caller(); //function graph trace的跳转失能
        FTRACE_WARN_ON(err);
}
```

**三、用户动态修改跟踪函数和命令**

**3.1 动态跟踪函数命令设置**

在支持动态trace的情况下，可以动态管理桩点，决定哪些函数的一级桩点跳转使能，可通过设置文件set\_ftrace\_filter/set\_ftrace\_notrace设置函数桩点。

设置方法为：echo "xxxx:xxx:xxx" \> set\_ftrace\_filter，其中echo写入的字段组织如下：

```
<function>:<command>:<parameter>
```

**function**：该字段是表明当前echo命令所作用的函数范围

**command**：该字段表明当前echo命令所要进行的动作，可选参数

        \- **mod�**�允许用户通过驱动模块来过滤跟踪函数

        \- **traceon**/**traceoff�**�允许用户跟踪某些函数到达一定计数后开启/关闭跟踪。不带参数则是命中即执行开启或关闭跟踪。（？看代码逻辑如果带参数是命中前n次时都开启或关闭跟踪，每次命中执行开启或关闭，n次后结束这种行为）

        \- **snapshot�**� 触发一个快照

        \- **enable\_event**/**disable\_event�**� 可以使能和失能函数的trace event

        \- **dump�**�dump the contents of the ftrace ring buffer to the console

        \- **cpudump �**�dump the contents of the ftrace ring buffer for the current CPU to the console

**paramter**：该字段作为command的参数传入，供其使用，可选参数。

例如：echo '\_\_schedule\_bug:traceoff:5' \> set\_ftrace\_filter

具体可以参考Documentation/trace/ftrace.txt文件描述

**3.2 只设置跟踪函数场景分析**

echo '\_\_schedule\_bug' \> set\_ftrace\_filter

echo '\!\_\_schedule\_bug' \> set\_ftrace\_filter

echo '\_\_schedule\*' \> set\_ftrace\_filter

echo '\_\_schedule\_bug' \> set\_ftrace\_notrace

echo '\!\_\_schedule\_bug' \> set\_ftrace\_notrace

echo '\_\_schedule\*' \> set\_ftrace\_notrace

相关文件file\_operations 为ftrace\_filter\_fops以及ftrace\_notrace\_fops

**3.2.1 文件open操作**

```
static int
ftrace_filter_open(struct inode *inode, struct file *file)
{
        //将文件关联到的ftrace_ops作为后续操作的基础，在tracing顶层目录下就是global_ops
        struct ftrace_ops *ops = inode->i_private; 
        return ftrace_regex_open(ops,
                        FTRACE_ITER_FILTER | FTRACE_ITER_DO_HASH,
                        inode, file);
}  
int ftrace_regex_open(struct ftrace_ops *ops, int flag,
                  struct inode *inode, struct file *file)
{
        if (file->f_mode & FMODE_WRITE) {
                const int size_bits = FTRACE_HASH_DEFAULT_BITS;
                //创建一份新的临时hash表实体，暂存至iter->hash，供后续写操作使用
                if (file->f_flags & O_TRUNC)
                        iter->hash = alloc_ftrace_hash(size_bits);
                else
                        iter->hash = alloc_and_copy_ftrace_hash(size_bits, hash);
        }
}
```

**3.2.2 文件write操作**

ftrace\_filter\_write\-\>ftrace\_regex\_write\-\>ftrace\_process\_regex

```
static int ftrace_process_regex(struct ftrace_hash *hash,
                                char *buff, int len, int enable)
{
        char *func, *command, *next = buff;
        struct ftrace_func_command *p;

        func = strsep(&next, ":");  //解析指定函数字段放入func中，剩余字段放入next中

        if (!next) {
                //如果next为空，则表明用户echo的参数中没有命令
                ret = ftrace_match_records(hash, func, len);
                return 0;
        }

                //运行到这说明用户echo的参数中有cmd，进一步解析将命令放入到command中，剩余命令参数放入next中
        command = strsep(&next, ":");
        list_for_each_entry(p, &ftrace_commands, list) {
                if (strcmp(p->name, command) == 0) {
                        ret = p->func(hash, func, command, next, enable);
                        goto out_unlock;
                }
        }

        return ret;
}
```

对\<function\>字段匹配并记录

```
//hash中表示文件对应的是用户所设置文件指向的ftrace_ops的filter或者是notrace链表，buff中存的是func字段
ftrace_match_records(struct ftrace_hash *hash, char *buff, int len)
{
        return match_records(hash, buff, len, NULL);
}
//mod是command的字段，此时mod入参为null，代码中mod相关的可不考虑
static int match_records(struct ftrace_hash *hash, char *func, int len, char *mod)
{
        struct ftrace_page *pg;
        struct dyn_ftrace *rec;
        struct ftrace_glob func_g = { .type = MATCH_FULL };
        int found = 0;
        int ret;
        int clear_filter = 0;

        if (func) {
                //filter_parse_regex对func字符进行正则方式解析，只解析！和*两种字符
                // * 符合决定匹配类型，并存在func_g.type中
                //1）MATCH_FULL  字符串全匹配
                //2）MATCH_END_ONLY 字符串尾部匹配
                //3）MATCH_FRONT_ONLY 字符串前端匹配
                //4）MATCH_MIDDLE_ONLY 组成中间匹配
                // ！决定过滤类型，存在clear_filter，！表示清除hash链表中的func表示的函数
                func_g.type = filter_parse_regex(func, len, &func_g.search, &clear_filter);
                func_g.len = strlen(func_g.search);
        }

        //遍历所有的dyn_ftrace
        do_for_each_ftrace_rec(pg, rec) { 

                if (rec->flags & FTRACE_FL_DISABLED)
                        continue;
                //先将地址通过kallsyms_lookup()转成函数名，在ftrace_match()函数中按类型进行匹配
                if (ftrace_match_record(rec, &func_g, NULL, { .type = MATCH_FULL })) {
                        //如果匹配到了，使用enter_record修改hash表
                        ret = enter_record(hash, rec, clear_filter);
                        found = 1;
                }
        } while_for_each_ftrace_rec();
out_unlock:
        return found;
}

static int enter_record(struct ftrace_hash *hash, struct dyn_ftrace *rec, int clear_filter)
{
        struct ftrace_func_entry *entry;
        int ret = 0;

        entry = ftrace_lookup_ip(hash, rec->ip);
        if (clear_filter) {
                /* Do nothing if it doesn't exist */
                if (!entry)
                        return 0;
                free_hash_entry(hash, entry);  //从hash链表中摘除并释放entry
        } else {
                /* Do nothing if it exists */
                if (entry)
                        return 0;
                ret = add_hash_entry(hash, rec->ip); //创建一个entry并加入hash链表
        }
        return ret;
}
```

**3.2.3 文件release操作**

```
int ftrace_regex_release(struct inode *inode, struct file *file)
{
        if (file->f_mode & FMODE_WRITE) {
                filter_hash = !!(iter->flags & FTRACE_ITER_FILTER);

                if (filter_hash)
                        orig_hash = &iter->ops->func_hash->filter_hash;
                else
                        orig_hash = &iter->ops->func_hash->notrace_hash;

                mutex_lock(&ftrace_lock);
                old_hash = *orig_hash;
                old_hash_ops.filter_hash = iter->ops->func_hash->filter_hash;
                old_hash_ops.notrace_hash = iter->ops->func_hash->notrace_hash;

                //将前面根据用户新echo的参数所创建的hash，更新到iter->ops->func_hash中，由于是使用rcu进行保护，相当于用副本替换
                ret =(iter->ops, filter_hash, orig_hash, iter->hash);
                if (!ret) {
                        //主要作用是更新一级桩点
                        ftrace_ops_update_code(iter->ops, &old_hash_ops);
                        free_ftrace_hash_rcu(old_hash);
                }
                mutex_unlock(&ftrace_lock);
        }
        free_ftrace_hash(iter->hash); //在ftrace_hash_move中已经使用完iter->hash，此处释放掉。
        kfree(iter);
}
```

**3.2.4 仅设置func字段总结**

对于只设置function这种场景，实质上只是更新文件指向的ops的filter\_hash/notrace\_hash链表，并根据链表修改一级桩点的指令值，以实现动态对某些函数的跟踪或不跟踪，达到动态跟踪的目的。

主要步骤：

1、将“func”字段表示的函数记录到hash链表中

2、根据hash链表更新一二级桩点指令

3、函数跟踪时即可调用trace\_ops的func方法跟踪用户想跟踪的具体函数

![09c0effaa51143a22ee125fe9487f7c6.png](image/09c0effaa51143a22ee125fe9487f7c6.png)

**3.2 存在cmd的场景分析**

以traceoff为例分析：echo '\_\_schedule\_bug:traceoff:5' \> set\_ftrace\_filter

其函数调用路径：

![da31418f935191b401249a2f49ff1a01.png](image/da31418f935191b401249a2f49ff1a01.png)

```
//入参glob是function字段，入参cmd是用户设置的command字段，param是用户设置的参数字段
static int ftrace_trace_onoff_callback(struct ftrace_hash *hash,
                            char *glob, char *cmd, char *param, int enable)
{
        struct ftrace_probe_ops *ops;
        //判断param参数是否存在，决定是否根据跟踪次数执行command
        if (strcmp(cmd, "traceon") == 0)
                ops = param ? &traceon_count_probe_ops : &traceon_probe_ops;
        else
                ops = param ? &traceoff_count_probe_ops : &traceoff_probe_ops;
        return ftrace_trace_probe_callback(ops, hash, glob, cmd,
                                           param, enable);
}

static int ftrace_trace_probe_callback(struct ftrace_probe_ops *ops,
                            struct ftrace_hash *hash, char *glob,
                            char *cmd, char *param, int enable)
{
        void *count = (void *)-1;
        char *number;
        int ret;

        if (glob[0] == '!') {
                unregister_ftrace_function_probe_func(glob+1, ops);
                return 0;
        }

        //参数param是一个数字，表示执行
        number = strsep(&param, ":"); 
        //将字符串形式的数字转换成unsigned long形式的数字，并存在count中。此处count为指针类型，实际上存的是数字。
        ret = kstrtoul(number, 0, (unsigned long *)&count); 

out_reg:
        ret = register_ftrace_function_probe(glob, ops, count);

        return ret < 0 ? ret : 0;
}
```

```
int register_ftrace_function_probe(char *glob, struct ftrace_probe_ops *ops, void *data)
{
    struct ftrace_hash **orig_hash = &trace_probe_ops.func_hash->filter_hash;
    struct ftrace_hash *old_hash = *orig_hash;
    

    //先将原先老的trace_probe_ops.func_hash->filter_hash表拷贝一份
    hash = alloc_and_copy_ftrace_hash(FTRACE_HASH_DEFAULT_BITS, old_hash);

    //遍历所有的函数桩点
    do_for_each_ftrace_rec(pg, rec) {

                //如果函数桩点与当前设置的func字段的函数不匹配，则进入下一个
                if (!ftrace_match_record(rec, &func_g, NULL, 0))
                        continue;
                
                //到这里，表明匹配到当前设置的func函数
                //申请一个struct ftrace_func_probe类型的entry
                entry = kmalloc(sizeof(*entry), GFP_KERNEL);
                count++;
                entry->data = data;

                if (ops->init) {
                        ops->init(ops, rec->ip, &entry->data)；
                }

                //将匹配到的桩点，记录到前面临时申请的hash链表中
                ret = enter_record(hash, rec, 0);

                            //在entry中记录ops函数和桩点地址，这两个值很重要
                entry->ops = ops;  
                entry->ip = rec->ip;

                key = hash_long(entry->ip, FTRACE_HASH_BITS);
                //将entry加入到全局的ftrace_func_hash链表中
                hlist_add_head_rcu(&entry->node, &ftrace_func_hash[key]);

        } while_for_each_ftrace_rec();

        //将前面记录了所有原始及新增桩点的hash链表，移到trace_probe_ops->func_hash->filter_hash中去
        ret = ftrace_hash_move(&trace_probe_ops, 1, orig_hash, hash);

        __enable_ftrace_function_probe(&old_hash_ops);
}

```

设置完成后的数据结构如下图所示，相比没有command字段的情况下多了一个trace\_probe\_ops，新增的trace\_probe\_ops主要用于控制commad关联函数执行。

1、系统运行时二级桩点函数ftrace\_ops\_list\_func\(\)会遍历ftrce\_ops\_list，根据ftrace\_ops中hash表中是否记录有当前函数，来判断是否执行ftrace\_ops\-\>func。因此需要在trace\_probe\_ops\-\>func\_hash中建立一个完整的桩点引用链表

2、在额外的ftrace\_func\_hash链表中，为附带有命令的函数申请一个struct ftrace\_func\_probe entry，用于记录具体函数所关联的命令及其操作方法

3、跟踪时，执行ftrace\_ops的func （function\_trace\_probe\_call），会最终调用到struct ftrace\_func\_probe\-\>ops

![b56efb3fdd90f7a53b6c910f126fc1ef.png](image/b56efb3fdd90f7a53b6c910f126fc1ef.png)

**四、二级桩函数执行**

如果ftrace\_ops\_list中注册有多个ops，那么二级桩函数会设置为ftrace\_ops\_list\_func（）

```
#define ftrace_ops_list_func ((ftrace_func_t)ftrace_ops_no_ops)
static void ftrace_ops_no_ops(unsigned long ip, unsigned long parent_ip)
{
        __ftrace_ops_list_func(ip, parent_ip, NULL, NULL);
}

static inline void
__ftrace_ops_list_func(unsigned long ip, unsigned long parent_ip,
                       struct ftrace_ops *ignored, struct pt_regs *regs)
{
        struct ftrace_ops *op;

        //遍历ftrace_ops_list链表中所有的ops
        do_for_each_ftrace_op(op, ftrace_ops_list) {
                if ((!(op->flags & FTRACE_OPS_FL_RCU) || rcu_is_watching()) &&
                    (!(op->flags & FTRACE_OPS_FL_PER_CPU) ||
                     !ftrace_function_local_disabled(op)) &&
                    ftrace_ops_test(op, ip, regs)) {  
            //ftrace_ops_test(op, ip, regs)测试ftrace_ops的hash表中是否有entry引用了函数桩点，引用了才能继续往下执行
                        op->func(ip, parent_ip, op, regs);
                }
        } while_for_each_ftrace_op(op);
        trace_clear_recursion(bit);
}
```

**五、参考资料**

1、https://www.cnblogs.com/hellokitty2/p/15624976.html

2、https://blog.csdn.net/pwl999/article/details/80702365/
