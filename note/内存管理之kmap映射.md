# 内存管理之kmap映射

vmalloc\(\)主要用于分配物理内存，并建立和虚拟地址的映射，如果已经分配好了物理内存（比如alloc\_page\(\)后），只是想建立映射，则可以使用vmap\(\)。vmap\(\)算是vmalloc\(\)的一个组成部分，但kmap\(\)跟kmalloc\(\)没有半毛钱关系。

通常呢，vmap\(\)用于相对长时间的映射，可同时映射多个pages，kmap\(\)和kmap\_atomic\(\)则用于相对短时间的映射，只能映射单个page。比如下面这个操作，就是将page cache（关于page cache的find\_get\_page的介绍请参考[这篇文�](https://zhuanlan.zhihu.com/p/68071761)�）中的某个page frame的内容初始化为0，很快就可以完成。

```
/* Find the page of interest. */
struct page *page = find_get_page(mapping, offset);

/* Gain access to the contents of that page. */
void *vaddr = kmap(page);

/* Do something to the contents of that page. */
memset(vaddr, 0, PAGE_SIZE);

/* Unmap that page. */
kunmap(vaddr);
```

给定物理页面后，kmap\(\)需要查找空余的虚拟地址空间，以建立两者之间的映射。那么这个查找过程具体是怎样的呢？在x86中，PKmap region虚拟地址空间的大小是4MB，而page的大小是4KB，因此一共可以容纳1024个pages（准确的说是建立1024个pages的映射）。

为什么设置为4MB大小呢，我想应该是这样1024个映射所需要的页表空间是1024\*4B = 4KB，刚好占一个page，不浪费。如果遍历这1024个pages，查找未被映射的，将会比较耗时。

想想vmalloc\(\)是怎么处理的呢？它用的是红黑树，相比于vmalloc/vmap，kmap只能映射一个page，且对应的虚拟地址空间更小，因而kmap采用的是更为简洁的hash table的形式来组织和查找。每个page的映射关系用一个**page\_address\_map**结构体表示，其中page指向对应物理页面的struct page，virtual是映射后的虚拟地址：

```
struct page_address_map {
        struct page *page;  //映射的物理页面
        void *virtual;  //映射的虚拟地址
        struct list_head list;  //加入hash链表中
};
```

一个hash bucket则用一个**page\_address\_slot**表示：

```
static struct page_address_slot {
        struct list_head lh;               /* List of page_address_maps */
        spinlock_t lock;                   /* Protect this bucket's list */
} ____cacheline_aligned_in_smp page_address_htable[1<<PA_HASH_ORDER];
```

page\_address\_map可通过list域挂载到page\_address\_slot中以lh域作为链表头的双向链表上。这里PA\_HASH\_ORDER的值是7，一个7阶的hash table有128个hash buckets，如果1024个映射全部建立的话，则一个page\_address\_slot上平均挂载1024/128=8个page\_address\_map 。

![fb1f1e948fcfeae4b3dab5f62f68849f.jfif](image/fb1f1e948fcfeae4b3dab5f62f68849f.jfif)

kmap中查找空闲虚拟地址的具体函数是page\_address\(\)。page\_address\(\)不仅可以查找一个high memory zone的物理页面通过kmap映射的虚拟地址，也可用于获取low memory zones（ZONE\_NORMAL/ZONE\_DMA，参考[这篇文�](https://zhuanlan.zhihu.com/p/68465952)�）的物理页面通过线性映射得到的虚拟地址。

lowmem\_page\_address\(\)的实现很简单，就是根据memory model，经过page\_to\_pfn\(\)转换得到struct page对应的物理地址（参考[这篇文�](https://zhuanlan.zhihu.com/p/68346347)�），加上偏移就是虚拟地址了。

```
void *lowmem_page_address(const struct page *page){
return page_to_virt(page);}

#define page_to_virt(x) __va(PFN_PHYS(page_to_pfn(x)))
#define __va(x) ((void *)((unsigned long)(x)+PAGE_OFFSET))
```

如果struct page对应的page frame是在high memory zone，则通过hash算法找到page frame所对应的page\_address\_slot，然后遍历page\_address\_slot上的page\_address\_map，根据page\_address\_map中的page域，查看是否有匹配的struct page，如果有的话，说明这个page之前建立过映射，那直接返回page\_address\_map中virtual域存储的虚拟地址就好了。

```
static struct page_address_slot *page_slot(const struct page *page)
{       
        return &page_address_htable[hash_ptr(page, PA_HASH_ORDER)];
}
void *page_address(const struct page *page)
{
        unsigned long flags;
        void *ret;
        struct page_address_slot *pas;

        if (!PageHighMem(page))
                return lowmem_page_address(page);

        pas = page_slot(page);  //通过page的地址计算hash至，并返回表头
        ret = NULL;
        spin_lock_irqsave(&pas->lock, flags);
        if (!list_empty(&pas->lh)) {
                struct page_address_map *pam;

                list_for_each_entry(pam, &pas->lh, list) {
                        if (pam->page == page) {  //遍历链表，如果page相等说明该page有映射到kmap，返回虚拟地址
                                ret = pam->virtual;
                                goto done;
                        }
                }
        }
done:
        spin_unlock_irqrestore(&pas->lock, flags);
        return ret;
}
```

如果没有找到呢，就要通过map\_new\_virtual\(\)建立新的映射了。这里get\_pkmap\_entries\_count\(\)是获取可以建立映射的最大数目，这个数目取决于PKmap region的大小，因此和具体的硬件体系有关（对于x86就是上面说的1024），color是用于某些架构的（比如MIPS和Xtensa），对于x86来说可以忽略。

```
static inline unsigned long map_new_virtual(struct page *page)
{
        unsigned long vaddr;
        int count;
        unsigned int last_pkmap_nr;
        unsigned int color = get_pkmap_color(page);

start:
        count = get_pkmap_entries_count(color);  //获取kmap entry 数量，通常是一个page所含有的pte表项数量 4096/4 = 1024
        /* Find an empty entry */
        for (;;) {
                last_pkmap_nr = get_next_pkmap_nr(color); //获取上一次kmap索引的下一个序号，序号一直在累加1
                if (no_more_pkmaps(last_pkmap_nr, color)) {
                        flush_all_zero_pkmaps();
                        count = get_pkmap_entries_count(color);
                }
                if (!pkmap_count[last_pkmap_nr])  //pkmap_count是一个位图，每个虚拟地址 kmap拥有一个bit表示是否映射，0表示未建立映射
                        break;  /* Found a usable entry */
                if (--count)
                        continue;

                /*
                 * Sleep for somebody else to unmap their entries
                 */
                //到这里表明kmap的虚拟地址全部用完了，只能休眠等待其他线程释放
                {
                        DECLARE_WAITQUEUE(wait, current);
                        wait_queue_head_t *pkmap_map_wait =
                                get_pkmap_wait_queue_head(color);

                        __set_current_state(TASK_UNINTERRUPTIBLE);
                        add_wait_queue(pkmap_map_wait, &wait);
                        unlock_kmap();  //spin_unlock_irq(&kmap_lock)  释放全局的kmap大锁，调度出去
                        schedule();
                        remove_wait_queue(pkmap_map_wait, &wait);
                        lock_kmap();

                        /* Somebody else might have mapped it while we slept */
                        if (page_address(page)) //在本线程休眠期间其他线程可能已经将该page映射了
                                return (unsigned long)page_address(page);

                        /* Re-start */
                        goto start;  //如果page还没有映射，则重新开始map
                }
        }
        vaddr = PKMAP_ADDR(last_pkmap_nr);  
        set_pte_at(&init_mm, vaddr,
                   &(pkmap_page_table[last_pkmap_nr]), mk_pte(page, kmap_prot));  //设置pte表项值，建立虚拟地址可物理地址的映射

        pkmap_count[last_pkmap_nr] = 1;  //标记该虚拟地址已经被映射
        set_page_address(page, (void *)vaddr);  //设置page_address_map,保存映射关系

        return vaddr;
}
```

pkmap\_count\[\]是一个位图，其数组长度等于PKmap region可以容纳的映射数目，数组中每个元素的值代表映射的状态，0代表映射未建立，也就是对应的虚拟地址可用。

get\_next\_pkmap\_nr\(\)使用了一个static局部变量last\_pkmap\_nr来存储上次建立映射的位置，以作为查找空闲虚拟地址的起点。因为系统启动后，按照分配规则，靠近PKmap region起始位置的虚拟地址会先被占用，从上次建立映射的位置开始查找，比从PKmap region的起始地址开始查找，通常能更快地找到空闲的虚拟地址。

如果找到了空闲的虚拟地址，就退出循环，调用set\_pte\_at\(\)填写内核页表，以建立这个虚拟地址和page frame的映射，并将pkmap\_count\[\]对应item的值设为1。set\_page\_address\(\)用于填写一个page\_address\_maps以存储这个映射关系，并将这个page\_address\_maps添加到page frame经过hash table得到的page\_address\_slot的链表上，方便以后查找。

![2e21b26a6b87323f3f1af168fea5bb48.jfif](image/2e21b26a6b87323f3f1af168fea5bb48.jfif)

如果遍历完pkmap\_count\[\]也没有找到空闲的虚拟地址，那就只能等待了，等待其他的线程解除映射，把坑让出来，这说明调用kmap\(\)是可能会sleep的。

整个过程用流程图表示大概是这样的：

![68808f689983e7a65170440c47b8e1fa.jfif](image/68808f689983e7a65170440c47b8e1fa.jfif)
