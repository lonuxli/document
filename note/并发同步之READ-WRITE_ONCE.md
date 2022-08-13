# 并发同步之READ/WRITE_ONCE

一、READ/WRITE\_ONCE定义

```
#define __READ_ONCE(x, check)                                           \
({                                                                      \
        union { typeof(x) __val; char __c[1]; } __u;                    \
        if (check)                                                      \
                __read_once_size(&(x), __u.__c, sizeof(x));             \
        else                                                            \
                __read_once_size_nocheck(&(x), __u.__c, sizeof(x));     \
        smp_read_barrier_depends(); /* Enforce dependency ordering from x */ \
        __u.__val;                                                      \
})
#define READ_ONCE(x) __READ_ONCE(x, 1)

#define __READ_ONCE_SIZE                                                \
({                                                                      \
        switch (size) {                                                 \
        case 1: *(__u8 *)res = *(volatile __u8 *)p; break;              \
        case 2: *(__u16 *)res = *(volatile __u16 *)p; break;            \
        case 4: *(__u32 *)res = *(volatile __u32 *)p; break;            \
        case 8: *(__u64 *)res = *(volatile __u64 *)p; break;            \
        default:                                                        \
                barrier();                                              \
                __builtin_memcpy((void *)res, (const void *)p, size);   \
                barrier();                                              \
        }                                                               \
})

static __always_inline
void __read_once_size(const volatile void *p, void *res, int size)
{
        __READ_ONCE_SIZE;
}
```

```
#define WRITE_ONCE(x, val) \
({                                                      \
        union { typeof(x) __val; char __c[1]; } __u =   \
                { .__val = (__force typeof(x)) (val) }; \
        __write_once_size(&(x), __u.__c, sizeof(x));    \
        __u.__val;                                      \
})

static __always_inline void __write_once_size(volatile void *p, void *res, int size)
{
        switch (size) {
        case 1: *(volatile __u8 *)p = *(__u8 *)res; break;
        case 2: *(volatile __u16 *)p = *(__u16 *)res; break;
        case 4: *(volatile __u32 *)p = *(__u32 *)res; break;
        case 8: *(volatile __u64 *)p = *(__u64 *)res; break;
        default:
                barrier();
                __builtin_memcpy((void *)p, (const void *)res, size);
                barrier();
        }
}
```

二、READ/WRITE\_ONCE使用场景

\* It makes code easier to understand

\* It is required by relevant standards

\* It enables automatic data race detection

\* It is required for kernel memory model

\* It may improve performance

三、READ/WRITE\_ONCE注意事项

四、参考资料

1、[https://lwn.net/Articles/508991/](https://lwn.net/Articles/508991/) \<ACCESS\_ONCE\(\)\>

2、[https://lwn.net/Articles/624126/](https://lwn.net/Articles/624126/)  \<ACCESS\_ONCE\(\) and compiler bugs\>

3、[https://zhuanlan.zhihu.com/p/102753962](https://zhuanlan.zhihu.com/p/102753962)

4、[https://github.com/google/kernel\-sanitizers/blob/master/other/READ\_WRITE\_ONCE.md](https://github.com/google/kernel-sanitizers/blob/master/other/READ_WRITE_ONCE.md)
