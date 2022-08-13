# 内存管理之slub

```
62 struct kmem_cache {
63 ▼       struct kmem_cache_cpu __percpu *cpu_slab; //一个per cpu变量，对于每个cpu来说，相当于一个本地内存缓存池。当分配内存的时候优先从本地cpu分配内存以保证cache的命中率。
64 ▼       /* Used for retriving partial slabs etc */
65 ▼       unsigned long flags;
66 ▼       unsigned long min_partial;
67 ▼       int size;▼      ▼       /* The size of an object including meta data */
68 ▼       int object_size;▼       /* The size of an object without meta data */
69 ▼       int offset;▼    ▼       /* Free pointer offset. */
70 ▼       int cpu_partial;▼       /* Number of per cpu partial objects to keep around */ //per cpu partial中所有slab的free object的数量的最大值，超过这个值就会将所有的slab转移到kmem_cache_node的partial链表。determined the maximum number of objects 决定本地cpu内存池中的obj数量    
71 ▼       struct kmem_cache_order_objects oo;
72
73 ▼       /* Allocation and freeing of slabs */
74 ▼       struct kmem_cache_order_objects max;
75 ▼       struct kmem_cache_order_objects min;
76 ▼       gfp_t allocflags;▼      /* gfp flags to use on each alloc */
77 ▼       int refcount;▼  ▼       /* Refcount for slab cache destroy */
78 ▼       void (*ctor)(void *);
79 ▼       int inuse;▼     ▼       /* Offset to metadata */
80 ▼       int align;▼     ▼       /* Alignment */
81 ▼       int reserved;▼  ▼       /* Reserved bytes at the end of slabs */
82 ▼       const char *name;▼      /* Name (only for display!) */
83 ▼       struct list_head list;▼ /* List of slab caches */
84 ▼       int red_left_pad;▼      /* Left redzone padding size */
111 ▼      struct kmem_cache_node *node[MAX_NUMNODES]; //slab节点。在NUMA系统中，每个node都有一个struct kmem_cache_node数据结构。
112 };

40 struct kmem_cache_cpu {                                                                                                            
41 ▼       void **freelist;▼       /* Pointer to next available object */ //指向下一个可用的object
42 ▼       unsigned long tid;▼     /* Globally unique transaction id */ //init_tid(cpu) 存放的是cpu序号
43 ▼       struct page *page;▼     /* The slab from which we are allocating */
44 ▼       struct page *partial;▼  /* Partially allocated frozen slabs */ //本地slab partial链表。
45 #ifdef CONFIG_SLUB_STATS
46 ▼       unsigned stat[NR_SLUB_STAT_ITEMS];
47 #endif
48 };

428 struct kmem_cache_node {
429 ▼       spinlock_t list_lock;
446 ▼       unsigned long nr_partial;
447 ▼       struct list_head partial;
455 };
```

SLUB 数据结构框图：

![](http://www.wowotech.net/content/uploadfile/201803/4a471520078976.png)
