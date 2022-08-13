# 内存管理之slab shrinker框架

slab shrinker用户回收小块的slab缓存，主要如dentry和inode缓存

基于4.9.37内核分析

**一、数据结构**

```
struct shrinker {
        unsigned long (*count_objects)(struct shrinker *,  
                                       struct shrink_control *sc); //return the number of freeable items in the cache
                                                                    //返回可回收的obj项
        unsigned long (*scan_objects)(struct shrinker *,
                                      struct shrink_control *sc);  // scan the cache and attempt to free items from the cache
                                                                //返回扫描过程中真正是否的obj数量

        int seeks;      /* seeks to recreate an obj */
                        // 《ULK》-informally, it is a parameter that indicates how much it costs to re-create one item of the cache once it is removed
                        // 它是一个参数，指示一旦删除缓存，重新创建缓存的一项会花费多少费用，用于调整扫描次数

        long batch;     /* reclaim batch size, 0 = default */ //回收批次大小,一次回收的大小
        unsigned long flags;

        /* These are for internal use */
        struct list_head list; //链接串起不同的shrinker
        /* objs pending delete, per node */
        atomic_long_t *nr_deferred;  //每个节点待删除的objs，应该是作为一个缓存，在某一node中上一次未完成的扫描次数，缓存住在下一次扫描累加
};
#define DEFAULT_SEEKS 2 /* A good number if you don't know better. */

struct shrink_control {
        gfp_t gfp_mask;

        /*
         * How many objects scan_objects should scan and try to reclaim.
         * This is reset before every call, so it is safe for callees
         * to modify.
         */
        unsigned long nr_to_scan;  //扫描的数量

        /* current node being shrunk (for NUMA aware shrinkers) */
        int nid;  //指定numa node id

        /* current memcg being shrunk (for memcg aware shrinkers) */
        struct mem_cgroup *memcg;  //指定mem cgroup
};
```

```
/*
* Add a shrinker callback to be called from the vm.
*/
int register_shrinker(struct shrinker *shrinker)
{       
        size_t size = sizeof(*shrinker->nr_deferred);
        
        if (shrinker->flags & SHRINKER_NUMA_AWARE)
                size *= nr_node_ids;
        
        shrinker->nr_deferred = kzalloc(size, GFP_KERNEL);
        if (!shrinker->nr_deferred)
                return -ENOMEM;
        
        down_write(&shrinker_rwsem);
        list_add_tail(&shrinker->list, &shrinker_list);
        up_write(&shrinker_rwsem);
        return 0;
}
EXPORT_SYMBOL(register_shrinker);

/*
* Remove one
*/
void unregister_shrinker(struct shrinker *shrinker)
{
        if (!shrinker->nr_deferred)
                return;
        down_write(&shrinker_rwsem);
        list_del(&shrinker->list);
        up_write(&shrinker_rwsem);
        kfree(shrinker->nr_deferred);
        shrinker->nr_deferred = NULL;
}
```

**二、slab回收**

```
static unsigned long shrink_slab(gfp_t gfp_mask, int nid,
                                 struct mem_cgroup *memcg,
                                 unsigned long nr_scanned,
                                 unsigned long nr_eligible)
{

    list_for_each_entry(shrinker, &shrinker_list, list) {
        freed += do_shrink_slab(&sc, shrinker, nr_scanned, nr_eligible); //循环遍历所有注册的shrinker回收
    }
}

static unsigned long do_shrink_slab(struct shrink_control *shrinkctl,
                                    struct shrinker *shrinker,
                                    unsigned long nr_scanned,
                                    unsigned long nr_eligible)
{ 
1、计算scan参数，为何scan计算如此复杂 ？
        freeable = shrinker->count_objects(shrinker, shrinkctl);
        nr = atomic_long_xchg(&shrinker->nr_deferred[nid], 0);

        total_scan = nr;
        delta = (4 * nr_scanned) / shrinker->seeks;
        delta *= freeable;
        do_div(delta, nr_eligible + 1);
        total_scan += delta;

//delta = (4 * nr_scanned * freeable) / (shrinker->seeks * nr_eligible)
//nr_scanned / nr_eligible 形成一个比例作为入参，调整扫描的数量

2、传入scan入参进行回收
        while (total_scan >= batch_size ||
               total_scan >= freeable) {
                unsigned long ret;
                unsigned long nr_to_scan = min(batch_size, total_scan);

                shrinkctl->nr_to_scan = nr_to_scan;
                ret = shrinker->scan_objects(shrinker, shrinkctl);
                if (ret == SHRINK_STOP)
                        break;
                freed += ret;

                count_vm_events(SLABS_SCANNED, nr_to_scan);
                total_scan -= nr_to_scan;
                scanned += nr_to_scan;

                cond_resched();
        }
}
```

**三、通用文件系统slab回收框架**

```
//文件系统super分配时注册都会注册一个通用的denry/inode shrinker
static struct super_block *alloc_super(struct file_system_type *type, int flags,
                                       struct user_namespace *user_ns)
{ 
       s->s_shrink.seeks = DEFAULT_SEEKS;
        s->s_shrink.scan_objects = super_cache_scan;
        s->s_shrink.count_objects = super_cache_count;
        s->s_shrink.batch = 1024;
        s->s_shrink.flags = SHRINKER_NUMA_AWARE | SHRINKER_MEMCG_AWARE;
}

static unsigned long super_cache_count(struct shrinker *shrink,
                                       struct shrink_control *sc)
{
        struct super_block *sb;
        long    total_objects = 0;

        sb = container_of(shrink, struct super_block, s_shrink);

        if (sb->s_op && sb->s_op->nr_cached_objects)
                total_objects = sb->s_op->nr_cached_objects(sb, sc); //统计其他slab cache

        total_objects += list_lru_shrink_count(&sb->s_dentry_lru, sc); //统计该文件系统dentry slab缓存数
        total_objects += list_lru_shrink_count(&sb->s_inode_lru, sc);  //统计该文件系统inode slab缓存数

        total_objects = vfs_pressure_ratio(total_objects);
        return total_objects;
}

static unsigned long super_cache_scan(struct shrinker *shrink,
                                      struct shrink_control *sc)
{
        struct super_block *sb;
        long    fs_objects = 0;
        long    total_objects;
        long    freed = 0;
        long    dentries;
        long    inodes;

        sb = container_of(shrink, struct super_block, s_shrink);

        /*
         * Deadlock avoidance.  We may hold various FS locks, and we don't want
         * to recurse into the FS that called us in clear_inode() and friends..
         */
        if (!(sc->gfp_mask & __GFP_FS))
                return SHRINK_STOP;

        if (!trylock_super(sb))
                return SHRINK_STOP;

        if (sb->s_op->nr_cached_objects)
                fs_objects = sb->s_op->nr_cached_objects(sb, sc);

        inodes = list_lru_shrink_count(&sb->s_inode_lru, sc);
        dentries = list_lru_shrink_count(&sb->s_dentry_lru, sc);
        total_objects = dentries + inodes + fs_objects + 1;
        if (!total_objects)
                total_objects = 1;

        /* proportion the scan between the caches */
        dentries = mult_frac(sc->nr_to_scan, dentries, total_objects);
        inodes = mult_frac(sc->nr_to_scan, inodes, total_objects);
        fs_objects = mult_frac(sc->nr_to_scan, fs_objects, total_objects);

        /*
         * prune the dcache first as the icache is pinned by it, then
         * prune the icache, followed by the filesystem specific caches
         *
         * Ensure that we always scan at least one object - memcg kmem
         * accounting uses this to fully empty the caches.
         */
        sc->nr_to_scan = dentries + 1;
        freed = prune_dcache_sb(sb, sc);
        sc->nr_to_scan = inodes + 1;
        freed += prune_icache_sb(sb, sc);

        if (fs_objects) {
                sc->nr_to_scan = fs_objects + 1;
                freed += sb->s_op->free_cached_objects(sb, sc);
        }

        up_read(&sb->s_umount);
        return freed;
}
```

**四、文件系统中其他slab cache回收框架**

```
struct super_operations {
        long (*nr_cached_objects)(struct super_block *,
                                  struct shrink_control *);
        long (*free_cached_objects)(struct super_block *,
                                    struct shrink_control *);
}
文件系统实现具体的nr_cached_objects和free_cached_objects，该框架用于回收除dentry和inode以外的文件系统特有的slab缓存。
例如：

static const struct super_operations xfs_super_operations = {
        .nr_cached_objects      = xfs_fs_nr_cached_objects,
        .free_cached_objects    = xfs_fs_free_cached_objects,
};

static const struct super_operations shmem_ops = {
#ifdef CONFIG_TRANSPARENT_HUGE_PAGECACHE
        .nr_cached_objects      = shmem_unused_huge_count,
        .free_cached_objects    = shmem_unused_huge_scan,
#endif
};
```

**五、vfs\_cache\_pressure �**�

控制文件系统中cache回收的压力

文件：/proc/sys/vm/vfs\_cache\_pressure

主要应用见函数super\_cache\_count\(\)，该参数控制扫描扫描百分比例，默认是100。

```
total_objects = vfs_pressure_ratio(total_objects);

static inline unsigned long vfs_pressure_ratio(unsigned long val)
{
        return mult_frac(val, sysctl_vfs_cache_pressure, 100);
}
```

**六、整体框架**

总共有三个入口进行slab回收，其中直接回收和周期回收的入参是不固定的，主动回收的入参固定

![85aaf0ea0bf1214a8acdefa7207f1ab3.png](image/85aaf0ea0bf1214a8acdefa7207f1ab3.png)

**七、4.15内核版本优化**

commit 9092c71bb724dba2ecba849eae69e5c9d39bd3d2

```
将扫描入参改为 priority，根据priority计算出扫描的数量：scan_objects = nr_objects >> sc->priority
static unsigned long shrink_slab(gfp_t gfp_mask, int nid,
                                 struct mem_cgroup *memcg,
                                 int priority)
{
                delta = freeable >> priority;
                delta *= 4;
                do_div(delta, shrinker->seeks);
}
```
