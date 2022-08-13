# linux网络管理之netlink组播框架

**一、背景**

从linux内核的patch日志来看，当前netlink组播的定义与此前2.6内核版本已发生变化，主要是对于一个特定的协议来说，组播组的个数可以由原先的32个变为可动态扩展。

**二、2.6内核版本**

```
struct sockaddr_nl
{
        sa_family_t     nl_family;      /* AF_NETLINK   */
        unsigned short  nl_pad;         /* zero         */
        __u32           nl_pid;         /* process pid  */  
        __u32           nl_groups;      /* multicast groups mask */
 };
```

**nl\_pid**：消息发往的目的id，在sockaddr\_nl

                sockaddr\_nl表示一个目的地址时，表示往该pid关联的sock发消息，

                sockaddr\_nl表示一个sock绑定的地址时，nl\_pid通常使用该线程/进程相关的pid，nl\_pid\+net代表了sock的唯一标记

**nl\_groups**：在2.6内核版本中，多播组掩码，因此最多32个组。

                sockaddr\_nl表示一个目的地址时，nl\_groups可以按照掩码的bit位表示多个组，即往这些组发送消息，即使当前源sock不在这个组，也能发消息

                sockaddr\_nl表示一个sock绑定的地址时，表示该sock加入所有bit位置1的组，当有组播消息发到这些组时，该sock都能接收到。

                

```
static int netlink_bind(struct socket *sock, struct sockaddr *addr, int addr_len)
{
        struct sock *sk = sock->sk;
        struct netlink_opt *nlk = nlk_sk(sk);
        struct sockaddr_nl *nladdr = (struct sockaddr_nl *)addr;
        int err;
        
        if (nladdr->nl_family != AF_NETLINK)
                return -EINVAL;

        /* Only superuser is allowed to listen multicasts */ //只有超级用户才能加入组播
        if (nladdr->nl_groups && !netlink_capable(sock, NL_NONROOT_RECV))
                return -EPERM;

        if (nlk->pid) {
                if (nladdr->nl_pid != nlk->pid)
                        return -EINVAL;
        } else {
                err = nladdr->nl_pid ? //绑定是nladdr有指定pid，则将该id和sock关联插入，如果nladdr中pid为0，则进行自动绑定分配id
                        netlink_insert(sk, nladdr->nl_pid) :
                        netlink_autobind(sock);
                if (err)
                        return err;
        }

        if (!nladdr->nl_groups && !nlk->groups)
                return 0;

        netlink_table_grab();
        if (nlk->groups && !nladdr->nl_groups)
        //如果本次绑定不支持组播，但是此sock之前支持组播，那么需要将sock从组播链表中摘除
                __sk_del_bind_node(sk); 
        else if (!nlk->groups && nladdr->nl_groups)
        //如果本次绑定支持组播，但是此sock之前没有支持组播并绑定，那么需要将sock加入组播链表
                sk_add_bind_node(sk, &nl_table[sk->sk_protocol].mc_list); 
        nlk->groups = nladdr->nl_groups;  //更新sock的组播，这里面保存的是mask，一个bit代表加入一个组播组中
        netlink_table_ungrab();

        return 0;
}
```

```
//netlink_broadcast作为消息广播函数，其中pid作为一个屏蔽ID，即广播时不往该ID发数据，通常在内核中用于屏蔽0，即代表内核的sock
int netlink_broadcast(struct sock *ssk, struct sk_buff *skb, u32 pid,
                      u32 group, int allocation)
{
        struct netlink_broadcast_data info;
        struct hlist_node *node;
        struct sock *sk;

        info.exclude_sk = ssk;
        info.pid = pid;  //目的id
        info.group = group;  //目的组
        info.failure = 0;
        info.congested = 0;
        info.delivered = 0;
        info.allocation = allocation;
        info.skb = skb;
        info.skb2 = NULL;

        sk_for_each_bound(sk, node, &nl_table[ssk->sk_protocol].mc_list)  //轮询protocol中所有接收组播信息的sock
                do_one_broadcast(sk, &info); //对轮询的sock尝试发送数据
}

static inline int do_one_broadcast(struct sock *sk,
                                   struct netlink_broadcast_data *p)
{
        struct netlink_opt *nlk = nlk_sk(sk);
        int val;

        //轮询的sock的pid不等于屏蔽的pid  && 目的组和轮询到的sock组有共同组，则继续发送消息
        //p->pid来源于netlink 发送时目的pid，在组播使用的情况下可以作为一个portid屏蔽 
        if (nlk->pid == p->pid || !(nlk->groups & p->group)) 
                goto out;

        if (p->failure) {
                netlink_overrun(sk);
                goto out;
        }

        sock_hold(sk);
        if (p->skb2 == NULL) {
                if (atomic_read(&p->skb->users) != 1) {
                        p->skb2 = skb_clone(p->skb, p->allocation);
                } else {
                        p->skb2 = p->skb;
                        atomic_inc(&p->skb->users);
                }
        }
        if (p->skb2 == NULL) {
                netlink_overrun(sk);
                /* Clone failed. Notify ALL listeners. */
                p->failure = 1;
        } else if ((val = netlink_broadcast_deliver(sk, p->skb2)) < 0) { //将需要发送的消息skb挂到目的sock的sk_receive_queue接收链表中
                netlink_overrun(sk);
        } else {
                p->congested |= val;
                p->delivered = 1;
                p->skb2 = NULL;
        }
        sock_put(sk);

out:
        return 0;
}  
```

**三、4.9内核版本**

为了保持兼容性，用户空间仍然可以使用bind订阅较低的32个组，并使用getsockname\(\)查看套接字订阅了哪些组，要订阅/取消订阅扩展范围内\(即32个之外\)的组，可以使用setsockopt\(\)来增加和剔除组。Struct nl\_addr最多只能包含32个组，通过struct sockaddr\_nl是没有办法指定超出32的组。

**3.1 创建sock并bind时的组订阅设置**

创建内核sock时先初始化组的宽度

```
struct sock *
__netlink_kernel_create(struct net *net, int unit, struct module *module,
                        struct netlink_kernel_cfg *cfg)
{
        if (!cfg || cfg->groups < 32)
                groups = 32; //组的宽度最小默认也为32
        else
                groups = cfg->groups;
        //listeners用于表明当前这个netlink协议中哪些组上有sock在监听，方便后续发送组播消息判断，没有监听这就不用遍历mc_list链表
        //因此listeners中存放组bit位的内存要与groups值一致，这个也应该是可扩展的。
        listeners = kzalloc(sizeof(*listeners) + NLGRPSZ(groups), GFP_KERNEL); 
        if (!nl_table[unit].registered) {
                nl_table[unit].groups = groups;
        }
}
```

创建具体sock时，初始化该sock相关组信息

```
static int netlink_bind(struct socket *sock, struct sockaddr *addr,
                        int addr_len)
{       
        //groups是从用户态传入的组信息，每个bit位表示一个组，最多32组
       long unsigned int groups = nladdr->nl_groups;

       if (groups) {
                if (!netlink_allowed(sock, NL_CFG_F_NONROOT_RECV))
                        return -EPERM;
                err = netlink_realloc_groups(sk);  //如果该sock有订阅组，按照nl_table[sk->sk_protocol].groups大小申请存放组bit位的内存
                if (err)
                        return err;
        }

        ...

        //sock没有低32组的订阅，退出，因为bind也只能设置低32组的订阅
        if (!groups && (nlk->groups == NULL || !(u32)nlk->groups[0]))
                return 0;

        netlink_table_grab();
        //nlk->subscriptions- hweight32(nlk->groups[0]) 是计算sock绑定设置前去掉低32组的订阅值
        //nlk->subscriptions- hweight32(nlk->groups[0]) + hweight32(groups)则是更新的订阅值，nlk->subscriptions中可能还保留大于32的组订阅信息
        netlink_update_subscriptions(sk, nlk->subscriptions +
                                         hweight32(groups) -
                                         hweight32(nlk->groups[0]));
        //更新低32组的订阅信息，将原先的低32bit清0，在把新传入的mask组信息按位或
        nlk->groups[0] = (nlk->groups[0] & ~0xffffffffUL) | groups;
        //该sock订阅的组可能是第一次被订阅，因此可能需要更新组监听bit位
        netlink_update_listeners(sk);
        netlink_table_ungrab();
}
```

**3.2 通过setsockopt\(\)设置大于32的组**

```
static int netlink_setsockopt(struct socket *sock, int level, int optname,
                              char __user *optval, unsigned int optlen)
{
        if (optlen >= sizeof(int) &&
            get_user(val, (unsigned int __user *)optval))
                return -EFAULT;

        switch (optname) {
        case NETLINK_ADD_MEMBERSHIP:
        case NETLINK_DROP_MEMBERSHIP: {
                if (!netlink_allowed(sock, NL_CFG_F_NONROOT_RECV))
                        return -EPERM;
        //存放组信息（一个bit代表一个组）是动态申请的，可能当前sock空间不够，需要将空间宽度拓展到nl_table[sk->sk_protocol].groups大小
                err = netlink_realloc_groups(sk);
                if (err)
                        return err;
        //此处val作为入参，表示的是要设置或清除的组播的index （从1开始）
        //如果大于nlk->ngroups表明该index超出当前支持的组播组容量，返回不让设置，nlk->ngroups存放的应该是组的宽度，位32的倍数，一开始应该就是32
                if (!val || val - 1 >= nlk->ngroups)  
                        return -EINVAL;
                              ...
                netlink_table_grab();
                //进行具体的组加入或删除处理
                netlink_update_socket_mc(nlk, val, optname == NETLINK_ADD_MEMBERSHIP);
                netlink_table_ungrab();
                if (optname == NETLINK_DROP_MEMBERSHIP && nlk->netlink_unbind)
                        nlk->netlink_unbind(sock_net(sk), val);

                err = 0;
                break;
        }
        }
}

static void netlink_update_socket_mc(struct netlink_sock *nlk,
                                     unsigned int group,
                                     int is_new)
{
        int old, new = !!is_new, subscriptions;

        old = test_bit(group - 1, nlk->groups);
        //计算出的subscriptions表明，经过本次加入组或取消组后，该sock是否还存在订阅组
        subscriptions = nlk->subscriptions - old + new;  
        if (new)
                __set_bit(group - 1, nlk->groups);  //加入group指定的组
        else
                __clear_bit(group - 1, nlk->groups); //退出group指定的组
        netlink_update_subscriptions(&nlk->sk, subscriptions); //更新
        netlink_update_listeners(&nlk->sk);
}

static void
netlink_update_subscriptions(struct sock *sk, unsigned int subscriptions)
{
        struct netlink_sock *nlk = nlk_sk(sk);

        if (nlk->subscriptions && !subscriptions)
                __sk_del_bind_node(sk); //如果此前是有订阅组，现在没有订阅，那么将sock从nl_table[sk->sk_protocol].mc_list中删除
        else if (!nlk->subscriptions && subscriptions)
                sk_add_bind_node(sk, &nl_table[sk->sk_protocol].mc_list); //相反此前没有现在有订阅，则将sock加入nl_table[sk->sk_protocol].mc_list
        nlk->subscriptions = subscriptions;
}
```

**3.3 组播消息发送时的组处理**

```
static int netlink_sendmsg(struct socket *sock, struct msghdr *msg, size_t len)
{
        DECLARE_SOCKADDR(struct sockaddr_nl *, addr, msg->msg_name);

        if (msg->msg_namelen) {
                dst_portid = addr->nl_pid;
                //addr->nl_groups是消息发往的目标组，这里ffs取最低一个bit位的序号值（1-32）作为dst_group，表明组播信息只能发送到一个组
                //而不能跟此前的2.6版本一样一下可以发到多个组，每个bit表示一个组
                dst_group = ffs(addr->nl_groups); 
        } else {
                dst_portid = nlk->dst_portid;
                dst_group = nlk->dst_group;
        }

        if (dst_group) {
                atomic_inc(&skb->users);
                netlink_broadcast(sk, skb, dst_portid, dst_group, GFP_KERNEL);
        }
        err = netlink_unicast(sk, skb, dst_portid, msg->msg_flags&MSG_DONTWAIT);
}

int netlink_broadcast(struct sock *ssk, struct sk_buff *skb, u32 portid,
                      u32 group, gfp_t allocation)
{
        return netlink_broadcast_filtered(ssk, skb, portid, group, allocation,
                NULL, NULL);
}  

int netlink_broadcast_filtered(struct sock *ssk, struct sk_buff *skb, u32 portid,
        u32 group, gfp_t allocation,
        int (*filter)(struct sock *dsk, struct sk_buff *skb, void *data),
        void *filter_data)
{
        info.portid = portid;
        info.group = group;

        sk_for_each_bound(sk, &nl_table[ssk->sk_protocol].mc_list) //遍历链表中每一个订阅加入了某个组的sock
                do_one_broadcast(sk, &info);
}

static void do_one_broadcast(struct sock *sk,
                                    struct netlink_broadcast_data *p)
{
        //
        if (nlk->portid == p->portid || p->group - 1 >= nlk->ngroups ||
            !test_bit(p->group - 1, nlk->groups)) //测试目的组号p->group - 1是否有设置
                return;
        ...
        
        netlink_broadcast_deliver(sk, p->skb2);  //skb数据发送处理
}
```

**四、参考资料：**

1、[https://lwn.net/Articles/147608/](https://lwn.net/Articles/147608/) 《Netlink: Support dynamic number of groups》

2、commit f7fa9b10edbb9391bdd4ec8e8b3d621d0664b198 《\[NETLINK\]: Support dynamic number of multicast groups per netlink family》
