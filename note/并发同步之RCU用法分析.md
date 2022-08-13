# 并发同步之RCU用法分析

**一、背景**

内核开发者使用了各种不同的同步原语来提升内核的并行性：细粒度锁（fine\-grained locks），无锁化数据结构（lock\-free data structures），CPU本地数据结构（per\-CPU data structures），以及读拷贝更新机制\(RCU\)。RCU机制在2002年开始在linux kernel中使用，在2013年已经超过6500个API在内核中被使用。RCU的成功得以与在并发读与写情况下的高性能，主要包括两个语义：读者在RCU读关键区间（RCU read\-side critical sections）中访问数据结构；写者使用RCU同步语句等待已在读关键区间的读者完成。

**二、RCU的好处**

RCU的存在满足了内核的三点需求，使得其在内核中应用广泛。

**2.1 在有写者的情况下支持并行读**

内核中有许多的数据结构被设计来支持大量的读写操作，特别是VFS以及网络子系统。

举个例子，VFS中使用dentry来缓存最近访问的文件元数据，因为应用程序会访问大量的文件，内核需要经常访问或者替换dentry，所以理想情况下线程在读取缓存数据的同时不希望与写冲突。RCU即满足了这种场景的需要。

**2.2 很低的计算与存储代价**

低空间代价是很重要的，因为内核会同时访问成千上万的对象，比如，在一些服务器中缓存了800万个dentry，就算在dentry的结构中加入极少的字节也会带来很大的存储代价。**低空间体现在内嵌的结构rcu\_head空间占用小**。

```
struct callback_head {
        struct callback_head *next;
        void (*func)(struct callback_head *head);
} __attribute__((aligned(sizeof(void *))));
#define rcu_head callback_head
```

低计算代价也很重要，因为内核频繁访问的数据结构往往是很短的代码路径，比如SELinux访问数据结构AVC，只有很短的代码路径进行访问，如果通过锁来保护，由于拿锁放锁带来的cache miss可能会有几百个cycle，而这就已经是访问AVC数据结构的一倍时间了，所以低执行代价在一些场景中非常重要。**低计算体现在rcu读端耗时微乎其微。**

**2.3 读操作确定的运行时间**

在NMI处理函数中访问共享数据的时候，线程可能在执行关键区域被中断。如果使用spinlock有可能带来死锁。而如果使用无锁化的数据结构，那么在冲突的时候就需要多次尝试直到成功，而这点却使得运行的时间变成非确定性的。

回到相关的一些同步原语设计：读写锁，本地锁，全局锁，事务性内存，都无法满足上诉讨论的特性。就算仅使用一个32位的字段存储锁数据，在很多场景下也是不可接受的。并且这些锁还会带来代价很大的原子操作以及内存屏障。

**三、RCU的设计**

RCU原语包含两个方面：RCU临界区以及RCU同步。

线程通过调用rcu\_read\_lock进入临界区，离开则是使用rcu\_read\_unlock。当需要进行RCU同步的时候，则使用synchronize\_rcu，当这个接口被调用之后，会等待所有的进入临界区的线程都退出了才会返回。synchronize\_rcu既不会阻止线程新进入临界区，也不会去等待synchronize\_rcu调用之后进入临界区的线程。**RCU允许线程去等待已在临界区的读者完成，但是不会提供多个写者之间的同步原语，多个写者还是需要锁机制来保证**。

为了满足读写并发、低空间低计算代价、以及读操作确定时间，Linux内核通过调度器上下文切换的契机进行RCU的同步处理。在进入临界区的时候禁止抢占，那么一次的上下文切换就意味着线程已经出了临界区。synchronize\_rcu则只需要等待所有的CPU经历一次上下文切换即可。

![Image.png](image/Image.png)

上述代码描述了一个RCU的简单实现。调用rcu\_read\_lock会禁止抢占并且是可以嵌套的。rcu\_read\_unlock允许抢占。为了保证所有的CPU都经历了一次上下文切换，线程在每个CPU上执行synchronize\_rcu。我们注意到执synchronize\_rcu的代价和执行rcu\_read\_lock与rcu\_read\_unlock的线程数目无关，只与CPU的个数有关。

在实际实现中，synchronize\_rcu会等待所有的CPU都经历过一次上下文切换，而不是去在每一个CPU上调度一下线程。

除了使用同步的synchronize\_rcu，线程还可以使用异步的原语call\_rcu，当所有的CPU都经历了上下文切换，则会通过一个回调函数来完成特定的需求。

由于RCU的读者和写者并行执行，在设计上还有需要考虑乱序问题，例如编译器指令乱序以及访存乱序。如果不能很好的保证顺序，读者有可能访问到一个写者初始化之前的值，造成空指针访问之类的问题。RCU通过接口rcu\_dereference和rcu\_assign\_pointer来保证访存顺序。这两个接口通过体系结构相关的一些内存屏障指令或者编译器指令来强制保证访存顺序。

**四、RCU的用法**

**4.1 rcu基本用法**

**4.1.1 数据结构定义**

使用案例：假定在一个四核cpu上有一个100000人管理系统，struct person表示人，其具有序号、年龄、财富、智商四个特征，cpu0/1/2 不间断找出具有某些特点的人，如年龄最小但财富最多的人。cpu3通过随机数生成修改系统中人员特征。使用rcu进行多核同步的代码写法：

```
struct person {
    struct rcu_head rcu;  //struct rcu_head嵌入对象中，方便后续回收时通过container_of拿到对象
    struct list_head list;
    int num; //序号
    int age; //年龄
    int money; //财富
    int iq; //智商
};
```

**4.1.2 多个读端**

cpu0\-cpu2每隔一段时间，遍历人员链表，cpu对person访问并非是原子性的，需要做保护，使用rcu\_read\_lock\(\)/rcu\_read\_unlock\(\)进行读保护。

```
struct list_head leader; //person链表，链表中初始化好了LIST_TEST_LEN个person
#define LIST_TEST_LEN 100000

void set_task_run_cpu(int cpu)
{
    struct cpumask mask;
    
    cpumask_clear(&mask);
    cpumask_set_cpu(cpu, &mask);
    if (set_cpus_allowed_ptr(current, &mask))
        printk("Set thread cpu:%d sched affinity failed!\n", cpu);
}

int get_data_cpu0(void *data)
{
    struct person *p;
    struct person r;
    int tmp = 0, max = 0;

    set_task_run_cpu(0);
    printk("enter thread cpu0\n");
    while(1) {
        rcu_read_lock();
        list_for_each_entry_rcu(p, &leader, list) {
            tmp = p->money / p->age;
            if (tmp > max) {
                memcpy(&r, p, sizeof(*p));
            }
        }
        rcu_read_unlock();
        printk("Most (money/age) person: %d %d %d %d", r.num, r.money, r.age, r.iq);
        max = 0;
        memset(&r, 0, sizeof(r));
        msleep(1000);
    }
    printk("exit thread cpu0\n");

    return 0;
}

int get_data_cpu1(void *data)
{
    struct person *p;
    struct person r;
    int tmp = 0, max = 0;

    set_task_run_cpu(1);
    printk("enter thread cp1\n");
    while(1) {
        rcu_read_lock();
        list_for_each_entry_rcu(p, &leader, list) {
            tmp = p->money / p->iq;
            if (tmp > max) {
                memcpy(&r, p, sizeof(*p));
            }
        }
        rcu_read_unlock();
        printk("Most (money/iq) person: %d %d %d %d", r.num, r.money, r.age, r.iq);
        max = 0;
        memset(&r, 0, sizeof(r));
        msleep(400);
    }
    printk("exit thread cpu1\n");

    return 0;
}

int get_data_cpu2(void *data)
{
    struct person *p;
    struct person r;
    int tmp = 0, max = 0;

    set_task_run_cpu(2);
    printk("enter thread cpu2\n");
    while(1) {
        rcu_read_lock();
        list_for_each_entry_rcu(p, &leader, list) {
            tmp = p->iq - p->age;
            if (tmp > max) {
                memcpy(&r, p, sizeof(*p));
            }
        }
        rcu_read_unlock();
        printk("Most (iq - age) person: %d %d %d %d", r.num, r.money, r.age, r.iq);
        max = 0;
        memset(&r, 0, sizeof(r));
        msleep(700);
    }
    printk("exit thread cpu2\n");

    return 0;
}
```

**4.1.3 写端**

cpu3 通过随机数修改person的数据，对person结构体的修改也不是原子性的，需要通过rcu做好多核同步。

- 同步回收

```
//旧的数据同步释放写法
int set_person_value_cpu3(void *data)
{
    struct person *p, *new;
    long refresh = 0;

    set_task_run_cpu(3);
    while(1) {
        get_random_bytes(&refresh, sizeof(refresh));
        refresh = refresh % LIST_TEST_LEN;
        list_for_each_entry(p, &leader, list) {
            if (refresh == p->num) {
                new = (struct person*)kzalloc(sizeof(*new), GFP_KERNEL);
                
                new->num = p->num;
                get_random_bytes(&refresh, sizeof(refresh));
                new->money = refresh;
                get_random_bytes(&refresh, sizeof(refresh));
                new->age =   refresh % 100;
                get_random_bytes(&refresh, sizeof(refresh));
                new->iq = refresh % 256;
                
                list_replace_rcu(&p->list, &new->list);
                synchronize_rcu(); //同步完成一次rcu宽限期后，再释放对象
                kfree(p);
                
                break;
            }
        }
        msleep(5);
    }
}
```

-  异步回收

```
//1）自定义回调函数异步释放写法
void free_person(struct rcu_head *head)
{
    struct person *p = NULL;
    p = container_of(head, struct person, rcu);
    kfree(p);
}

int set_person_value_cpu3(void *data)
{
    struct person *p, *new;
    long refresh = 0;

    set_task_run_cpu(3);
    while(1) {
        get_random_bytes(&refresh, sizeof(refresh));
        refresh = refresh % LIST_TEST_LEN;
        list_for_each_entry(p, &leader, list) {
            if (refresh == p->num) {
                new = (struct person*)kzalloc(sizeof(*new), GFP_KERNEL);
                new->num = p->num;
                get_random_bytes(&refresh, sizeof(refresh));
                new->money = refresh;
                get_random_bytes(&refresh, sizeof(refresh));
                new->age =   refresh % 100;
                get_random_bytes(&refresh, sizeof(refresh));
                new->iq = refresh % 256;
                
                list_replace_rcu(&p->list, &new->list);
                call_rcu(&p->rcu, free_person); //写回调函数异步回收
                
                break;
            }
        }
        msleep(5);
    }
}

//2）利用封装好的kfree_rcu函数，直接异步释放
int set_person_value_cpu3(void *data)
{
    struct person *p, *new;
    long refresh = 0;

    set_task_run_cpu(3);
    while(1) {
        get_random_bytes(&refresh, sizeof(refresh));
        refresh = refresh % LIST_TEST_LEN;
        list_for_each_entry(p, &leader, list) {
            if (refresh == p->num) {
                new = (struct person*)kzalloc(sizeof(*new), GFP_KERNEL);
                new->num = p->num;
                get_random_bytes(&refresh, sizeof(refresh));
                new->money = refresh;
                get_random_bytes(&refresh, sizeof(refresh));
                new->age =   refresh % 100;
                get_random_bytes(&refresh, sizeof(refresh));
                new->iq = refresh % 256;
      
                list_replace_rcu(&p->list, &new->list);
                kfree_rcu(p, rcu); //利用封装好的kfree释放接口释放slab
                
                break;
            }
        }
        msleep(5);
    }
}
```

**4.1.4 kfree\_rcu机制**

内核中数据结构一般是使用kmalloc分配，使用kfree释放，如果针对每一个kmalloc分配的内存对象，且只需释放该数据结构本身，都实现一个rcu回调内存释放函数，代码显得比较冗余。能否利用这类对象的操作相似性，使用相同的回调函数覆盖只需释放内存结构的场景？kfree\_rcu就是满足这种需求的，其实现原理：

1、数据结构对象定义时，需要将struct rcu\_head嵌入其中。

2、调用kfree\_rcu时需要传入对象地址以及struct rcu\_head类型成员名，以计算struct rcu\_head类型成员在数据结构对象中的偏移。

3、调用call\_rcu进行异步回收时，传入的func并非是真的函数地址，而是struct rcu\_head类型成员在数据结构对象中的偏移

4、在宽限期完成之后，进行回收执行\_\_rcu\_reclaim函数时，判断“func”是偏移还是真正的回调函数，如果是偏移，则使用kfree释放内存。考虑到使用该机制的内核数据结构通常是小于4096字节的，因此判断“func”是偏移还是真正的回调函数就是将其值与4096做比较。

```
#define kfree_rcu(ptr, rcu_head)                    \
    __kfree_rcu(&((ptr)->rcu_head), offsetof(typeof(*(ptr)), rcu_head))

#define __is_kfree_rcu_offset(offset) ((offset) < 4096)

#define __kfree_rcu(head, offset) \
    do { \
        BUILD_BUG_ON(!__is_kfree_rcu_offset(offset)); \
        kfree_call_rcu(head, (rcu_callback_t)(unsigned long)(offset)); \
    } while (0)

void kfree_call_rcu(struct rcu_head *head,
            rcu_callback_t func)
{
    __call_rcu(head, func, rcu_state_p, -1, 1);
}

static inline bool __rcu_reclaim(const char *rn, struct rcu_head *head)
{
    unsigned long offset = (unsigned long)head->func;

    rcu_lock_acquire(&rcu_callback_map);
    if (__is_kfree_rcu_offset(offset)) {
        RCU_TRACE(trace_rcu_invoke_kfree_callback(rn, head, offset));
        kfree((void *)head - offset); //通过kfree 释放slab内存
        rcu_lock_release(&rcu_callback_map);
        return true;
    } else {
        RCU_TRACE(trace_rcu_invoke_callback(rn, head));
        head->func(head); //正常调用rcu 回调函数
        rcu_lock_release(&rcu_callback_map);
        return false;
    }
}
```

**4.1.5 多个写端**

假如还有个cpu4 要更新链表怎么办？需要使用额外的锁对链表进行保护，实现多个写之间的互斥。

```
int set_person_value_cpu3(void *data)
{
    struct person *p, *new;
    long refresh = 0;

    set_task_run_cpu(3);
    while(1) {
        get_random_bytes(&refresh, sizeof(refresh));
        refresh = refresh % LIST_TEST_LEN;
        spin_lock(&list_lock);//==>用锁对多个写链表进行互斥，list_replace_rcu中对链表的修改需要多步，写重入会造成链表异常因此需要额外锁保护
        list_for_each_entry(p, &leader, list) {
            if (refresh == p->num) {
                new = (struct person*)kzalloc(sizeof(*new), GFP_KERNEL);
                ... //set new
                
                list_replace_rcu(&p->list, &new->list);
                spin_unlock(&list_lock);
                kfree_rcu(p, rcu); //利用封装好的kfree释放接口释放slab
                break;
            }
        }
        spin_unlock(&list_lock);//==>用锁对多个写链表进行互斥
        msleep(5);
    }
}

int set_person_value_cpu4(void *data)
{
    struct person *p, *new;
    long refresh = 0;

    set_task_run_cpu(4);
    while(1) {
        get_random_bytes(&refresh, sizeof(refresh));
        refresh = refresh % LIST_TEST_LEN;
        spin_lock(&list_lock); //==>用锁对多个写链表进行互斥
        list_for_each_entry(p, &leader, list) {
            if (refresh == p->num) {
                new = (struct person*)kzalloc(sizeof(*new), GFP_KERNEL);
                ... //set new
                
                list_replace_rcu(&p->list, &new->list);
                spin_unlock(&list_lock);
                kfree_rcu(p, rcu); //利用封装好的kfree释放接口释放slab
                break;
            }
        }
        spin_unlock(&list_lock);//==>用锁对多个写链表进行互斥
        msleep(5);
    }
}
```

**4.2 读写锁的替代�**�

RCU最常见的用途是替换读写锁。在20世纪90年代初期，Paul在实现通用RCU之前，实现了一种轻量级的读写锁。后来，为这个轻量级读写锁原型所设想的每个用途，最终都使用RCU来实现了。 RCU和读写锁最关键的相似之处，在于两者都有可以并行执行读端临界区。事实上，在某些情况下，完全可以用对应的读写锁API来替换RCU的API，反之亦然。 RCU的优点在于：性能、不会死锁，以及良好的实时延迟。当然RCU也有一点缺点，比如：读者与更新者并发执行，低优先级RCU读者也可以阻塞正等待优雅周期结束的高优先级线程，优雅周期的延迟可能达到好几毫秒。

**相同点：**

1、两者都有可以并行执行读端临界区；写端需互斥。

**不同点：**

1、rcu读端性能好、以及良好的实时延迟，多核扩展性好。（下图读端临界区长度为0时，多核扩展场景下读端开销，rcu几乎就是水平扩展，因为其读端只有开关抢占，所以时间延迟也是确定的）

**![rcu_rwlock_cpu_e.png](image/rcu_rwlock_cpu_e.png)**

2、rcu读者与更新者并发执行，低优先级RCU读者也可以阻塞正等待优雅周期结束的高优先级线程。优先级反转。

3、rcu读端原语基本上是不会死锁的，因为它本身就属于无锁编程的范畴。

4、RCU读端可能读到的不是最新的数据，读写锁读端读到的必然是最新数据。（RCU中，在更新者完成后才开始的读者都“保证”能看见新值，在更新者开始后才完成的读者有可能看见新值，也有可能看见旧值，这取决于具体的时机。）

代码示例：使用RCU作为读写锁的一个例子是同步访问PID哈希表：

```
pid_table_entry_t pid_table[];
process_t *pid_lookup(int pid)
{
    process_t *p;
    rcu_read_lock();
    p = pid_table[pid_hash(pid)].process;
    if (p)
    atomic_inc(&p->ref);
    rcu_read_unlock();
    return p;
}
void pid_free(process *p)
{
    if (atomic_inc(&p->ref))
    free(p);
}
void pid_remove(int pid)
{
    process_t **p;
    spin_lock(&pid_table[pid_hash(pid)].lock);
    p = &pid_table[pid_hash(pid)].process;
    rcu_assign_pointer(p, NULL);
    spin_unlock(&pid_table[pid_hash(pid)].lock);
    if (*p)
    call_rcu(pid_free, *p);
}
```

**4.3 受限的引用计数**

rcu当做一个简单受限的引用计数使用，读端临界区表明对对象的引用。

使用时计数：

```
1  rcu_read_lock();                /*  acquire  reference.  */
2  p  =  rcu_dereference(head);
3  /*  do  something  with  p.  */
4  rcu_read_unlock();            /*  release  reference.  */
```

计数为0时释放：

```
1  spin_lock(&mylock);
2  p  =  head;
3  rcu_assign_pointer(head,  NULL);
4  spin_unlock(&mylock);
5  /*  Wait  for  all  references  to  be  released.  */
6  synchronize_rcu();
7  kfree(p);
```

_实际案例：_下面的伪代码展示了RCU在Linux网络栈中取代引用计数的应用。该例子利用RCU对IP options进行引用计数，表示当前内核网络栈正在拷贝IP options到一个packet中。udp\_sendmsg调用rcu\_read\_lock与rcu\_read\_unlock标记临界区的进入与退出。而应用程序可以通过系统调用sys\_setsockop\(最终到内核的setsockopt函数\)对IP options进行修改。修改的过程是先将新值拷贝，然后调用call\_rcu进行旧值的删除。

```
void udp_sendmsg(sock_t *sock, msg_t *msg)
{
    ip_options_t *opts;
    char packet[];
    copy_msg(packet, msg);
    rcu_read_lock();
    opts = rcu_dereference(sock->opts);
    if (opts != NULL)
    copy_opts(packet, opts);
    rcu_read_unlock();
    queue_packet(packet);
}
void setsockopt(sock_t *sock, int opt, void *arg)
{
    if (opt == IP_OPTIONS) {
        ip_options_t *old = sock->opts;
        ip_options_t *new = arg;
        rcu_assign_pointer(&sock->opts, new);
        if (old != NULL)
        call_rcu(kfree, old);
        return;
    }
    /* Handle other opt values */
}
```

**4.4 批量引用计数机制**

略~

**4.5 穷人版的垃圾回收器**

RCU与GC有几点不同：

（1）程序员必须手动指示何时可以回收指定数据结构。（对象需要使用者调用call\_rcu\(\)指示回收）

（2）程序员必须手动标出可以合法持有引用的RCU读端临界区。（高级语言new了对象后可以不受限使用）

**4.6 存在担保**

略~

**4.7 类型安全的内存**

类型内存安全：是指某一块内存不再使用了或者要被释放掉，但依然能保证在一段周期内这块内存所存数据类型不会发生变化，且能当做一个正常的该类型使用。

**4.7.1 信号的锁问题**

进程信号处理使用task\_struct\-\>sighand，在多核场景下和exec\(\)存在竞争，exec\(\)中可能会修改task\_struct\-\>sighand的指针值，考虑下如何使用锁来进行互斥？？？

```
struct task_struct {
    struct sighand_struct *sighand;
}

struct sighand_struct {
    atomic_t        count;
    struct k_sigaction  action[_NSIG];
    spinlock_t      siglock;
    wait_queue_head_t   signalfd_wqh;
};
```

**4.7.2 早期2.6内核方案**

早先的send\_sigqueue\(\)信号处理函数，使用tsk\-\>sighand的锁在tsk\-\>sighand\-\>siglock中（应该是考虑小粒度的锁性能更好），如果想正常锁住tsk\-\>sighand\-\>siglock的话，外部需要一把大锁tasklist\_lock来锁住，因为读取tsk\-\>sighand和锁住tsk\-\>sighand\-\>siglock不是原子的，可能在两者之间被中断或抢占修改锁位置的内存数据，如果这期间tsk\-\>sighand改变了，那么将出现上锁错误，因此此处需要使用tasklist\_lock大锁。

```
//写端写tsk->sighand
static inline int de_thread(struct task_struct *tsk)
{
    struct sighand_struct *newsighand, *oldsighand = tsk->sighand;
    
    //创建一个新的sighand
    newsighand = kmem_cache_alloc(sighand_cachep, GFP_KERNEL);
    spin_lock_init(&newsighand->siglock);
    atomic_set(&newsighand->count, 1);
    memcpy(newsighand->action, oldsighand->action,
                 sizeof(newsighand->action));
  
    write_lock_irq(&tasklist_lock);
    spin_lock(&oldsighand->siglock);
    spin_lock(&newsighand->siglock);
  
    current->sighand = newsighand; //写端更新
    recalc_sigpending();
  
    spin_unlock(&newsighand->siglock);
    spin_unlock(&oldsighand->siglock);
    write_unlock_irq(&tasklist_lock);
  
    if (atomic_dec_and_test(&oldsighand->count))
         kmem_cache_free(sighand_cachep, oldsighand);
}

//读端读tsk->sighand
int send_sigqueue(int sig, struct sigqueue *q, struct task_struct *tsk)  
{
    read_lock(&tasklist_lock);
    spin_lock_irqsave(&tsk->sighand->siglock, flags);
    
    //进行具体的信号处理
    
    spin_unlock_irqrestore(&tsk->sighand->siglock, flags);
    read_unlock(&tasklist_lock);
}
```

**4.7.3 rcu优化方案**

引入rcu后，在rcu临界区内通过保证读取到的struct sighand\_struct类型的tsk\-\>sighand\-\>siglock的类型安全，即内存中数据类型不变， 多次尝试实现正确上锁，以优化性能。而这个临界区内类型安全就是依赖rcu以及SLAB\_DESTROY\_BY\_RCU实现的。

1）创建slab cache时带入SLAB\_DESTROY\_BY\_RCU标志，该标志的作用是，在内核free掉这类slab内存时，不会立即释放掉，而是将其通过call\_rcu回调进行延迟释放，如果在rcu读临界区内调用，则真正的释放至少要在退出读临界区后。

```
void __init proc_caches_init(void)
{
    sighand_cachep = kmem_cache_create("sighand_cache",
            sizeof(struct sighand_struct), 0,
            SLAB_HWCACHE_ALIGN|SLAB_PANIC|SLAB_DESTROY_BY_RCU|
            SLAB_NOTRACK|SLAB_ACCOUNT, sighand_ctor);
}
static void free_slab(struct kmem_cache *s, struct page *page)
{
    if (unlikely(s->flags & SLAB_DESTROY_BY_RCU)) {
        struct rcu_head *head;
        head = &page->rcu_head;
        call_rcu(head, rcu_free_slab);  //如果有SLAB_DESTROY_BY_RCU通过rcu回收
    } else   
        __free_slab(s, page);
}   
```

2）在写端更新task\_struct\-\>sighand时，先创建一个新的数据，使用rcu\_assign\_pointer进行发布。

```
//写端 多核场景下exec()时可能会修改task_struct->sighand，与使用的地方产生竞争
static int de_thread(struct task_struct *tsk)
{
        newsighand = kmem_cache_alloc(sighand_cachep, GFP_KERNEL);

        atomic_set(&newsighand->count, 1);
        memcpy(newsighand->action, oldsighand->action,
               sizeof(newsighand->action));

        write_lock_irq(&tasklist_lock);
        spin_lock(&oldsighand->siglock);
        rcu_assign_pointer(tsk->sighand, newsighand); //写发布
        spin_unlock(&oldsighand->siglock);
        write_unlock_irq(&tasklist_lock);

        __cleanup_sighand(oldsighand); //kfree(oldsighand); 释放旧数据
}
```

3）读端通过重试上锁

```
truct sighand_struct *__lock_task_sighand(struct task_struct *tsk,
                       unsigned long *flags)
{
    struct sighand_struct *sighand;

    for (;;) {
        /*
         * Disable interrupts early to avoid deadlocks.
         * See rcu_read_unlock() comment header for details.
         */
        local_irq_save(*flags);
        rcu_read_lock();
        sighand = rcu_dereference(tsk->sighand);

//==》1）这个空档可能有写端把tsk->sighand修改了，但是旧的old_sighand肯定未释放掉
        spin_lock(&sighand->siglock);
        if (likely(sighand == tsk->sighand)) {
            rcu_read_unlock();
            break;
            //==》如果没有写端修改，认为拿锁成功，则以spin_lock锁住的状态返回
        }
//==》2）到这来说明有写端修改了tsk->sighand，但是并未释放，此时的sighand已经是没有用的      
        spin_unlock(&sighand->siglock);
        rcu_read_unlock();
        local_irq_restore(*flags);
//==>3)进入下一次for循环，尝试获取tsk->sighand->siglock
    }

    return sighand;
}
```

**4.7.4 参考commit**

1、\<\[PATCH\] RCU signal handling\> e56d090310d7625ecb43a1eeebd479f04affb48b

2、\<\[PATCH\] convert sighand\_cache to use SLAB\_DESTROY\_BY\_RCU\> aa1757f90bea3f598b6e5d04d922a6a60200f1da

**4.8 等待事务结束**

**4.8.1 NMI中断问题**

试想下案例：不可屏蔽中断处理函数nmi\_profile\(\)读取内存buf并做修改（操作二级指针buf\-\>entry\[x\]修改内存），nmi\_stop\(\)函数停止前者对这块内存的操作。

```
struct  profile_buffer  {
     long  size;
     atomic_t  entry[0];
};

static  struct  profile_buffer  *buf  =  NULL;

//NMI中断调用 =》读端
void  nmi_profile(unsigned  long  pcvalue)
{
    atomic_inc(&buf->entry[pcvalue]);
}

//进程上下文调用 =》写端
void  nmi_stop(void)
{
    struct  profile_buffer p = buf;
    buf = NULL;
    kfree(p);
}
```

**问题难点：**

1、nmi\_profile\(\)中对内存的访问需要使用二级指针，不能让nmi\_stop\(\)简单对buf置空并释放，多核并行情况下可能出现nmi\_profile\(\)内存的访问异常。

2、不能使用常规的spin\_lock\_irq\(\)/spin\_unlock\_irq\(\)锁保护，由于关中断关不了NMI中断，单核场景下nmi\_profile\(\)重入nmi\_stop\(\)会造成死锁。

3、使用其他方案如原子变量或锁会麻烦些......

**3.8.1 常规方案**

使用原子变量保护需要考虑的会较多，代码可能也会显得复杂，相当于实现一个简单的读端不会被阻塞的读写锁。

1、读端需要并行，读端和写端需要互斥

2、在单核上，读端抢占了处于临界区的写端，要能return。也就是读端发现写端上锁了，需要直接返回不能直接自旋。

3、读端如果发现写端上锁了，需要自旋等待

```
//NMI中断调用 =》读端
void  nmi_profile(unsigned  long  pcvalue)
{
    //1、如果写端上锁，直接return；2、如果读端上锁，直接进入并行
    if (buf == NULL)
        return
    atomic_inc(&buf->entry[pcvalue]);
    //3、读端解锁
}

//进程上下文调用 =》写端
void  nmi_stop(void)
{
    struct  profile_buffer p;
    
    //1、如果写端上锁，自旋等待 2、如果写端上锁，自旋等待
    p = buf;
    buf = NULL;
    //3、写端解锁
    if (p)
        kfree(p);
}
```

**4.8.2 rcu解决方案**

rcu可以巧妙并且简单的解决该问题：

```
//读端：在NMI中断中通过二级指针这种非原子操作
void  nmi_profile(unsigned  long  pcvalue)
{
    struct profile_buffer *p = rcu_dereference(buf);

    if  (p  ==  NULL)
         return;
    if  (pcvalue  >=  p->size)
         return;
    atomic_inc(&p->entry[pcvalue]);
}

//写端，先将buf置空，经历至少一个rcu临界区后，NMI中断中对buf的引用必然结束。
//此时可以将原先buf指向的资源释放掉，NMI本身处于中断临界区，没有必要使用rcu_read_lock/unlock标记
void  nmi_stop(void)
{
    struct  profile_buffer  *p  =  buf;

    if  (p  ==  NULL)
        return;
    rcu_assign_pointer(buf,  NULL);
    synchronize_sched();
    kfree(p);
}
```

**4.8.3 类似案例防止NMI下死锁**

```
rcu_list_t nmi_list;
spinlock_t nmi_list_lock;
void handle_nmi()
{
    rcu_read_lock();
    rcu_list_for_each(&nmi_list, handler_t cb)
    cb();
    rcu_read_unlock();
}
void register_nmi_handler(handler_t cb)
{
    spin_lock(&nmi_list_lock);
    rcu_list_add(&nmi_list, cb);
    spin_unlock(&nmi_list_lock);
}
void unregister_nmi_handler(handler_t cb)
{
    spin_lock(&nmi_list_lock);
    rcu_list_remove(cb);
    spin_unlock(&nmi_list_lock);
    synchronize_rcu();
}
```

上面的伪代码展示了NMI系统的工作流程nmi\_list保存了NMI的处理函数，并且使用spinlock进行写保护，但支持无锁化的读操作。rcu\_list\_for\_each在遍历每一个元素的时候会调用rcu\_dereference，rcu\_list\_ad和rcu\_list\_remove则会调用rcu\_assign\_pointer。在一个RCU临界区内，NMI系统会执行所有的NMI处理函数。注销处理函数的时候，NMI系统会先清空list，然后调用synchronize\_rcu保证它返回时所有的处理函数都已经完成了。

在此场景下，如果想使用读写锁，很容易造成死锁，CPU在unregister\_nmi\_handler中拿锁的情况下，依然会被NMI打断，NMI处理函数中也会尝试拿锁，造成死锁。

在使用rcu以前，读端（调用NMI处理函数）和写端（注册NMI处理函数），都使用spin\_lock\_irq，由于NMI中断不可屏蔽的特征，导致可能会出现死锁的情况。使用RCU以后，不存在死锁的情况。
