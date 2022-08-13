# 内存管理之非LRU可移动页面

**一、什么是no\-lru moveable页面**

用于zram, GPU memory等的页面，内核本身不会直接使用这些页面地址，页面内容可以移动。

如果一个驱动想让自己的页面可被移动，需要实现address\_space\_operations中的以下方法。

//隔离页面，为no\-lru moveable页面迁移功能新增函数

bool \(\*isolate\_page\) \(struct page \*page, isolate\_mode\_t mode\);

//迁移页面，原先就有，有些文件系统实现了自身的migratepage函数

int \(\*migratepage\) \(struct address\_space \*mapping,

                    struct page \*newpage, struct page \*oldpage, enum migrate\_mode\);

//如果迁移失败，将被隔离的页面返回驱动，为no\-lru moveable页面迁移功能新增函数

void \(\*putback\_page\)\(struct page \*\);

**二、no\-lru movable页面的标志**

通过page\-\>mapping判断页面是否是no\-lru moveable页面，并

\#define PAGE\_MAPPING\_ANON       0x1

\#define PAGE\_MAPPING\_MOVABLE    0x2

\#define PAGE\_MAPPING\_KSM        \(PAGE\_MAPPING\_ANON | PAGE\_MAPPING\_MOVABLE\)

\#define PAGE\_MAPPING\_FLAGS      \(PAGE\_MAPPING\_ANON | PAGE\_MAPPING\_MOVABLE\)

//判断页面是否是no\-lru movable页面

static \_\_always\_inline int \_\_PageMovable\(struct page \*page\)

{

        return \(\(unsigned long\)page\-\>mapping & PAGE\_MAPPING\_FLAGS\) ==

                                PAGE\_MAPPING\_MOVABLE;

}

//判断页面是否是no\-lru movable页面并具备相应的迁移操作方法

int PageMovable\(struct page \*page\)

{

        struct address\_space \*mapping;

        VM\_BUG\_ON\_PAGE\(\!PageLocked\(page\), page\);

        if \(\!\_\_PageMovable\(page\)\)

                return 0;

        mapping = page\_mapping\(page\);

        if \(mapping && mapping\-\>a\_ops && mapping\-\>a\_ops\-\>isolate\_page\)

                return 1;

        return 0;

}

![Image.png](image/Image.png)

处于被隔离状态的no\-lru moveable页面，会被设置PG\_isolated标志，代码中没看到设置。 

enum pageflags {

        PG\_reclaim,             /\* To be reclaimed asap \*

        /\* non\-lru isolated movable page \*/

        PG\_isolated = PG\_reclaim,

}

PageIsolated\(page\)  //判断是否隔离

\_\_SetPageIsolated\(page\);  //设置隔离标志

\_\_ClearPageIsolated\(page\);  //清除隔离标志

**三、no\-lru moveable迁移实现**

1、isolate 页面

static unsigned long

isolate\_migratepages\_block\(struct compact\_control \*cc, unsigned long low\_pfn,

                        unsigned long end\_pfn, isolate\_mode\_t isolate\_mode\)

{

                /\*

                 \* Check may be lockless but that's ok as we recheck later.

                 \* It's possible to migrate LRU and non\-lru movable pages.

                 \* Skip any other type of page

                 \*/

                if \(\!PageLRU\(page\)\) {

                        /\*

                         \* \_\_PageMovable can return false positive so we need

                         \* to verify it under page\_lock.

                         \*/

                        if \(unlikely\(\_\_PageMovable\(page\)\) &&

                                        \!PageIsolated\(page\)\) { 

                        //如果是no\-lru moveable页面且未隔离，则进行隔离，其他返回失败

                                if \(locked\) {

                                        spin\_unlock\_irqrestore\(zone\_lru\_lock\(zone\),

                                                                        flags\);

                                        locked = false;

                                }

                                if \(isolate\_movable\_page\(page, isolate\_mode\)\)

                                        goto isolate\_success;

                        }

                        goto isolate\_fail;

                }

}

bool isolate\_movable\_page\(struct page \*page, isolate\_mode\_t mode\)

{

        struct address\_space \*mapping;

        /\*

         \* Avoid burning cycles with pages that are yet under \_\_free\_pages\(\),

         \* or just got freed under us.

         \*

         \* In case we 'win' a race for a movable page being freed under us and

         \* raise its refcount preventing \_\_free\_pages\(\) from doing its job

         \* the put\_page\(\) at the end of this block will take care of

         \* release this page, thus avoiding a nasty leakage.

         \*/

        if \(unlikely\(\!get\_page\_unless\_zero\(page\)\)\)

                goto out;

        /\*

         \* Check PageMovable before holding a PG\_lock because page's owner

         \* assumes anybody doesn't touch PG\_lock of newly allocated page

         \* so unconditionally grapping the lock ruins page's owner side.

         \*/

        if \(unlikely\(\!\_\_PageMovable\(page\)\)\)

                goto out\_putpage;

        /\*

         \* As movable pages are not isolated from LRU lists, concurrent

         \* compaction threads can race against page migration functions

         \* as well as race against the releasing a page.

         \*

         \* In order to avoid having an already isolated movable page

         \* being \(wrongly\) re\-isolated while it is under migration,

         \* or to avoid attempting to isolate pages being released,

         \* lets be sure we have the page lock

         \* before proceeding with the movable page isolation steps.

         \*/

        if \(unlikely\(\!trylock\_page\(page\)\)\)

                goto out\_putpage;

        if \(\!PageMovable\(page\) || PageIsolated\(page\)\)

                goto out\_no\_isolated;

        mapping = page\_mapping\(page\);

        VM\_BUG\_ON\_PAGE\(\!mapping, page\);

        if \(\!mapping\-\>a\_ops\-\>isolate\_page\(page, mode\)\)

                goto out\_no\_isolated;

        /\* Driver shouldn't use PG\_isolated bit of page\-\>flags \*/

        WARN\_ON\_ONCE\(PageIsolated\(page\)\);

        \_\_SetPageIsolated\(page\);

        unlock\_page\(page\);

        return true;

out\_no\_isolated:

        unlock\_page\(page\);

out\_putpage:

        put\_page\(page\);

out:

        return false;

}

2、migratepage页面：

static int move\_to\_new\_page\(struct page \*newpage, struct page \*page,

                                enum migrate\_mode mode\)

{

        struct address\_space \*mapping;

        int rc = \-EAGAIN;

        bool is\_lru = \!\_\_PageMovable\(page\);  //判断该页面是否是no\-lru moveable页面，也可用认为是否在lru链表中

        VM\_BUG\_ON\_PAGE\(\!PageLocked\(page\), page\);

        VM\_BUG\_ON\_PAGE\(\!PageLocked\(newpage\), newpage\);

        mapping = page\_mapping\(page\);

        if \(likely\(is\_lru\)\) {  //lru链表页面 普通文件页\+匿名页面\+在swap space中匿名页面

                if \(\!mapping\)

                        //匿名页面迁移

                        rc = migrate\_page\(mapping, newpage, page, mode\);

                else if \(mapping\-\>a\_ops\-\>migratepage\)

                        /\*

                         \* Most pages have a mapping and most filesystems

                         \* provide a migratepage callback. Anonymous pages

                         \* are part of swap space which also has its own

                         \* migratepage callback. This is the most common path

                         \* for page migration.

                         \*/

                        //部分文件系统和swap space实现了自身的migratepage方法，则调用其迁移方法

                        rc = mapping\-\>a\_ops\-\>migratepage\(mapping, newpage,

                                                        page, mode\);

                else

                        //Default handling if a filesystem does not provide a migration function.

                        rc = fallback\_migrate\_page\(mapping, newpage,

                                                        page, mode\);

        } else { //非lru链表页面

                /\*

                 \* In case of non\-lru page, it could be released after

                 \* isolation step. In that case, we shouldn't try migration.

                 \*/

                VM\_BUG\_ON\_PAGE\(\!PageIsolated\(page\), page\);

                if \(\!PageMovable\(page\)\) { //如果不可移动，直接退出

                        rc = MIGRATEPAGE\_SUCCESS;

                        \_\_ClearPageIsolated\(page\);

                        goto out;

                }

                //如果可移动，则调用该页面对应的移动方法进行迁移

                rc = mapping\-\>a\_ops\-\>migratepage\(mapping, newpage,

                                                page, mode\);

                WARN\_ON\_ONCE\(rc == MIGRATEPAGE\_SUCCESS &&

                        \!PageIsolated\(page\)\);

        }

        ...

}

3、putback\_page页面：

/\* It should be called on page which is PG\_movable \*/

void putback\_movable\_page\(struct page \*page\)

{

        struct address\_space \*mapping;

        VM\_BUG\_ON\_PAGE\(\!PageLocked\(page\), page\);

        VM\_BUG\_ON\_PAGE\(\!PageMovable\(page\), page\);

        VM\_BUG\_ON\_PAGE\(\!PageIsolated\(page\), page\);

        mapping = page\_mapping\(page\);

        mapping\-\>a\_ops\-\>putback\_page\(page\);

        \_\_ClearPageIsolated\(page\);

}

put\_back策略：

                if \(unlikely\(\_\_PageMovable\(page\)\)\) {

                        VM\_BUG\_ON\_PAGE\(\!PageIsolated\(page\), page\);

                        lock\_page\(page\);

                        if \(PageMovable\(page\)\)

                                putback\_movable\_page\(page\); //no\-lru moveable页面put back

                        else

                                \_\_ClearPageIsolated\(page\);

                        unlock\_page\(page\);

                        put\_page\(page\);

                } else {

                        dec\_node\_page\_state\(page, NR\_ISOLATED\_ANON \+

                                        page\_is\_file\_cache\(page\)\);

                        putback\_lru\_page\(page\); //普通lru页面put back

                }

**四、no\-lru moveable应用案例**

当系统支持规整时，zram实现了页面迁移的方法

\#ifdef CONFIG\_COMPACTION

static int zs\_register\_migration\(struct zs\_pool \*pool\);

\#else

tatic int zs\_register\_migration\(struct zs\_pool \*pool\) { return 0; }

\#endif

static int zs\_register\_migration\(struct zs\_pool \*pool\)

{

        pool\-\>inode = alloc\_anon\_inode\(zsmalloc\_mnt\-\>mnt\_sb\);

        if \(IS\_ERR\(pool\-\>inode\)\) {

                pool\-\>inode = NULL;

                return 1;

        }

        pool\-\>inode\-\>i\_mapping\-\>private\_data = pool;

        pool\-\>inode\-\>i\_mapping\-\>a\_ops = &zsmalloc\_aops;

        return 0;

}

static const struct address\_space\_operations zsmalloc\_aops = {

        .isolate\_page = zs\_page\_isolate,

        .migratepage = zs\_page\_migrate,

        .putback\_page = zs\_page\_putback,

};

**五、参考资料6**

commit id：bda807d4445414e8e77da704f116bb0880fe0c7

zRAM相关资料

[http://www.wowotech.net/memory\_management/zram.html](http://www.wowotech.net/memory_management/zram.html)
