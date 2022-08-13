# linux网络管理之内核通知链

基于linux4.9

**一、通知链背景**

Linux内核中各个子系统相互依赖，当其中某个子系统状态发生改变时，就必须使用一定的机制告知使用其服务的其他子系统，以便其他子系统采取相应的措施。为满足这样的需求，内核实现了事件通知链机制（notificationchain）。通知链只能用在各个子系统之间，而不能在内核和用户空间进行事件的通知

**二、数据结构**

notifier\_fn\_t是通知链发生时回调函数的形式

notifier\_block是通知链中元素的基本形式

```
typedef int (*notifier_fn_t)(struct notifier_block *nb,
                        unsigned long action, void *data);
                        
struct notifier_block {
        notifier_fn_t notifier_call;
        struct notifier_block __rcu *next;
        int priority;
};
```

**三、基本概念**

1.  原子通知链（ Atomic notifier chains ）：通知链元素的回调函数（当事件发生时要执行的函数）在中断或原子操作上下文中运行，不允许阻塞。对应的链表头结构：

```
struct atomic_notifier_head {
        spinlock_t lock;
        struct notifier_block __rcu *head;
};
```

2.  可阻塞通知链（ Blocking notifier chains ）：通知链元素的回调函数在进程上下文中运行，允许阻塞。对应的链表头：

```
struct blocking_notifier_head {
        struct rw_semaphore rwsem;
        struct notifier_block __rcu *head;
};
```

3.     原始通知链（ Raw notifierchains ）：对通知链元素的回调函数没有任何限制，所有锁和保护机制都由调用者维护。对应的链表头：

```
struct raw_notifier_head {
        struct notifier_block __rcu *head;
};
```

4、srcu 通知链，对应的表头：

```
struct srcu_notifier_head {
        struct mutex mutex;
        struct srcu_struct srcu;
        struct notifier_block __rcu *head;
};
```

&为何要设计出不同的通知链形式？

&为何不同的通知链需要使用不一样的锁？

以atomic形式的通知链为例，通知链的注册与注销都用了spin\_lock\_irqsave保护起来，这样可以在中断上下文使用注册与注销函数。

在通知链的调用函数\_\_atomic\_notifier\_call\_chain中，通过rcu\_read\_lock/unlock将通知链链表作为rcu读端遍历保护起来了，而注册注销则作为rcu写端。对于注册来讲，本质上是往链表中添加元素，不影响读端，因此只需使用rcu发布（rcu\_assign\_pointer）即可；对于注销断则是摘除链表元素，因此需要使用synchronize\_rcu等待宽限期完成，但是我们可以注意到，宽限期完成之后没有资源的释放，因为通知链元素基本是静态定义的，如果是非静态的可以在注销之后释放。

```
int atomic_notifier_chain_register(struct atomic_notifier_head *nh,
                struct notifier_block *n)
{
        unsigned long flags;
        int ret;

        spin_lock_irqsave(&nh->lock, flags);
        ret = notifier_chain_register(&nh->head, n);
        spin_unlock_irqrestore(&nh->lock, flags);
        return ret;
}

int atomic_notifier_chain_unregister(struct atomic_notifier_head *nh,
                struct notifier_block *n)
{
        unsigned long flags;
        int ret;

        spin_lock_irqsave(&nh->lock, flags);
        ret = notifier_chain_unregister(&nh->head, n);
        spin_unlock_irqrestore(&nh->lock, flags);
        synchronize_rcu();  //注销断等待宽限期完成
        return ret;
}

int __atomic_notifier_call_chain(struct atomic_notifier_head *nh,
                                 unsigned long val, void *v,
                                 int nr_to_call, int *nr_calls)
{
        int ret;

        rcu_read_lock();
        ret = notifier_call_chain(&nh->head, val, v, nr_to_call, nr_calls);
        rcu_read_unlock();
        return ret;
}
```

**四、基本的使用形式**

1、定义一个通知链表头，封装出模块专属的通知事件注册/注销函数，其本质上也就是向该链表头指向的链表中添加元素。

```
static ATOMIC_NOTIFIER_HEAD(netevent_notif_chain);

int register_netevent_notifier(struct notifier_block *nb)
{
        return atomic_notifier_chain_register(&netevent_notif_chain, nb);
}

int unregister_netevent_notifier(struct notifier_block *nb)
{
        return atomic_notifier_chain_unregister(&netevent_notif_chain, nb);
}
```

2、封装出一个模块专有的通知链调用函数

```
int call_netevent_notifiers(unsigned long val, void *v)
{
        return atomic_notifier_call_chain(&netevent_notif_chain, val, v);
}
```

3、定义一个通知链处理元素，并静态赋值回调函数，注册进对应的模块事件通知链（本质上就是加入该链的链表中）

如下可以看到，回调处理函数中基本形式是通过switch判断入参（即通知链调用时传入的参数）来做想应的动作。

```
static int nb_callback(struct notifier_block *self, unsigned long event,
                       void *ctx)
{
        switch (event) {
        case (NETEVENT_NEIGH_UPDATE):{
                cxgb_neigh_update((struct neighbour *)ctx);
                break;
        }
        case (NETEVENT_REDIRECT):{
                struct netevent_redirect *nr = ctx;
                cxgb_redirect(nr->old, nr->new, nr->neigh,
                              nr->daddr);
                cxgb_neigh_update(nr->neigh);
                break;
        }
        default:
                break;
        }
        return 0;
}

static struct notifier_block nb = {
        .notifier_call = nb_callback
};

register_netevent_notifier(&nb);
```

4、在需要的地方调用通知链，以达到事件通知的目的

```
call_netevent_notifiers(NETEVENT_DELAY_PROBE_TIME_UPDATE, p);
```
