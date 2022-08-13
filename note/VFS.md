# VFS

```
inode.c

//根据super_block + inode编号计算hash值
static inline unsigned long hash(struct super_block *sb, unsigned long hashval)
{
        unsigned long tmp;

        tmp = (hashval * (unsigned long)sb) ^ (GOLDEN_RATIO_PRIME + hashval) /
                        L1_CACHE_BYTES;
        tmp = tmp ^ ((tmp ^ GOLDEN_RATIO_PRIME) >> I_HASHBITS);
        return tmp & I_HASHMASK;
}

//根据super_block + inode编号获取哈希链表
struct hlist_head *head = inode_hashtable + hash(sb, ino);
```
