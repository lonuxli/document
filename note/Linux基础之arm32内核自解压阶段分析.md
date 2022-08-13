# Linux基础之arm32内核自解压阶段分析

假定内核镜像在物理地址0x42000000

arch/arm/boot/compressed/head.S

uboot跳转至start运行

r0：未使用 r1：存放architecture ID  r2：atags的物理地址

```
start:
        .type    start,#function
        .rept    7
        __nop
        .endr
   ARM(        mov    r0, r0        )
   ARM(        b    1f        )
THUMB(        badr    r12, 1f        )
THUMB(        bx    r12        )

        .word    _magic_sig    @ Magic numbers to help the loader
        .word    _magic_start    @ absolute load/run zImage address
        .word    _magic_end    @ zImage end address
        .word    0x04030201    @ endianness flag

THUMB(        .thumb            )
1:        __EFI_HEADER

ARM_BE8(    setend    be        )    @ go BE8 if compiled for BE8
AR_CLASS(    mrs    r9, cpsr    )
#ifdef CONFIG_ARM_VIRT_EXT
        bl    __hyp_stub_install    @ get into SVC mode, reversibly
#endif
        mov    r7, r1            @ save architecture ID   //r7=arc id
        mov    r8, r2            @ save atags pointer     //r8=atags addr

#ifndef CONFIG_CPU_V7M
        /*
         * Booting from Angel - need to enter SVC mode and disable
         * FIQs/IRQs (numeric definitions from angel arm.h source).
         * We only do this if we were in user mode on entry.
         */
        mrs    r2, cpsr        @ get current mode
        tst    r2, #3            @ not user?
        bne    not_angel
        mov    r0, #0x17        @ angel_SWIreason_EnterSVC
ARM(        swi    0x123456    )    @ angel_SWI_ARM
THUMB(        svc    0xab        )    @ angel_SWI_THUMB
not_angel:
        safe_svcmode_maskall r0
        msr    spsr_cxsf, r9        @ Save the CPU boot mode in
                        @ SPSR
#endif
        /*
         * Note that some cache flushing and other stuff may
         * be needed here - is there an Angel SWI call for this?
         */

        /*
         * some architecture specific code can be inserted
         * by the linker here, but it should preserve r7, r8, and r9.
         */

        .text

#ifdef CONFIG_AUTO_ZRELADDR
        /*
         * Find the start of physical memory.  As we are executing
         * without the MMU on, we are in the physical address space.
         * We just need to get rid of any offset by aligning the
         * address.
         *
         * This alignment is a balance between the requirements of
         * different platforms - we have chosen 128MB to allow
         * platforms which align the start of their physical memory
         * to 128MB to use this feature, while allowing the zImage
         * to be placed within the first 128MB of memory on other
         * platforms.  Increasing the alignment means we place
         * stricter alignment requirements on the start of physical
         * memory, but relaxing it means that we break people who
         * are already placing their zImage in (eg) the top 64MB
         * of this range.
         */
        mov    r4, pc    //r4=0x420000a8
        and    r4, r4, #0xf8000000   //r4=0x40000000(phys)
        /* Determine final kernel image address. */
        add    r4, r4, #TEXT_OFFSET  //r4=0x40008000(phys)
#else
        ldr    r4, =zreladdr
#endif

        /*
         * Set up a page table only if it won't overwrite ourself.
         * That means r4 < pc || r4 - 16k page directory > &_end.
         * Given that r4 > &_end is most unfrequent, we add a rough
         * additional 1MB of room for a possible appended DTB.
         */
        mov    r0, pc
        cmp    r0, r4          //r0=0x420000b4(phys)    r4=0x40008000(phys)  
        ldrcc    r0, LC0+32    
        addcc    r0, r0, pc
        cmpcc    r4, r0
        orrcc    r4, r4, #1        @ remember we skipped cache_on
        blcs    cache_on

restart:    adr    r0, LC0
        ldmia    r0, {r1, r2, r3, r6, r10, r11, r12}  // r0=0x42000248(phys) r1=0x248
        ldr    sp, [r0, #28] //sp=0x0035aad8

        /*
         * We might be running at a different address.  We need
         * to fix up various pointers.
         */
        sub    r0, r0, r1        @ calculate the delta offset  //r0=0x42000000  即自解压程序运行时的物理地址与链接地址的差值
        add    r6, r6, r0        @ _edata    //r6=0x42359ab8(phys) _edata/压缩镜像结束物理地址
        add    r10, r10, r0        @ inflated kernel size location  //r10=0x42359a7b(phys)  该地址存放压缩前镜像大小

        /*
         * The kernel build system appends the size of the
         * decompressed kernel at the end of the compressed data
         * in little-endian form.
         */
        ldrb    r9, [r10, #0]
        ldrb    lr, [r10, #1]
        orr    r9, r9, lr, lsl #8
        ldrb    lr, [r10, #2]
        ldrb    r10, [r10, #3]
        orr    r9, r9, lr, lsl #16
        orr    r9, r9, r10, lsl #24   //读取并计算压缩前镜像大小r9=0x00924000

#ifndef CONFIG_ZBOOT_ROM
        /* malloc space is above the relocated stack (64k max) */
        add    sp, sp, r0        //sp=0x4235aad8(phys)
        add    r10, sp, #0x10000  //r10=0x4236aad8(phys)
#else
        /*
         * With ZBOOT_ROM the bss/stack is non relocatable,
         * but someone could still run this code from RAM,
         * in which case our reference is _edata.
         */
        mov    r10, r6
#endif

/*
* Check to see if we will overwrite ourselves.
*   r4  = final kernel address (possibly with LSB set)  //解压后存放内核镜像起始地址 r4=0x40008000(phys)
*   r9  = size of decompressed image                    //解压后内核镜像大小 r9=0x00924000
*   r10 = end of this image, including  bss/stack/malloc space if non XIP  //自解压程序栈结束地址 r10=0x4236aad8(phys)
* We basically want:
*   r4 - 16k page directory >= r10 -> OK
*   r4 + image length <= address of wont_overwrite -> OK
* Note: the possible LSB in r4 is harmless here.
*/
        add    r10, r10, #16384
        cmp    r4, r10
        bhs    wont_overwrite          //r4 - 16k >= r10  自解压程序、栈及zImage镜像在解压后存放的内核镜像之前，不会出现写覆盖
        add    r10, r4, r9             //r10=0x4092c000(phys) 解压后内核镜像结束地址
        adr    r9, wont_overwrite      //r9=ZIMAGE_ADDR+0x188(phys)
        cmp    r10, r9
        bls    wont_overwrite          // 解压后内核镜像结束地址在当前运行的程序wont_overwrite之前，不会出现写覆盖

/*
* Relocate ourselves past the end of the decompressed kernel.
*   r6  = _edata
*   r10 = end of the decompressed kernel
* Because we always copy ahead, we need to do it from the end and go
* backward in case the source and destination overlap.
*/
        /*
         * Bump to the next 256-byte boundary with the size of
         * the relocation code added. This avoids overwriting
         * ourself when the offset is small.
         */
        add    r10, r10, #((reloc_code_end - restart + 256) & ~255)
        bic    r10, r10, #255

        /* Get start of code we want to copy and align it down. */
        adr    r5, restart
        bic    r5, r5, #31

/* Relocate the hyp vector base if necessary */
#ifdef CONFIG_ARM_VIRT_EXT
        mrs    r0, spsr
        and    r0, r0, #MODE_MASK
        cmp    r0, #HYP_MODE
        bne    1f

        bl    __hyp_get_vectors
        sub    r0, r0, r5
        add    r0, r0, r10
        bl    __hyp_set_vectors
1:
#endif

        sub    r9, r6, r5        @ size to copy
        add    r9, r9, #31        @ rounded up to a multiple
        bic    r9, r9, #31        @ ... of 32 bytes
        add    r6, r9, r5
        add    r9, r9, r10

1:        ldmdb    r6!, {r0 - r3, r10 - r12, lr}
        cmp    r6, r5
        stmdb    r9!, {r0 - r3, r10 - r12, lr}
        bhi    1b

        /* Preserve offset to relocated code. */
        sub    r6, r9, r6

#ifndef CONFIG_ZBOOT_ROM
        /* cache_clean_flush may use the stack, so relocate it */
        add    sp, sp, r6
#endif

        bl    cache_clean_flush

        badr    r0, restart
        add    r0, r0, r6
        mov    pc, r0

wont_overwrite:
/*
* If delta is zero, we are running at the address we were linked at.
*   r0  = delta
*   r2  = BSS start
*   r3  = BSS end
*   r4  = kernel execution address (possibly with LSB set)
*   r5  = appended dtb size (0 if not present)
*   r7  = architecture ID
*   r8  = atags pointer
*   r11 = GOT start
*   r12 = GOT end
*   sp  = stack pointer
*/
        orrs    r1, r0, r5
        beq    not_relocated

        add    r11, r11, r0
        add    r12, r12, r0

#ifndef CONFIG_ZBOOT_ROM
        /*
         * If we're running fully PIC === CONFIG_ZBOOT_ROM = n,
         * we need to fix up pointers into the BSS region.
         * Note that the stack pointer has already been fixed up.
         */
        add    r2, r2, r0
        add    r3, r3, r0

        /*
         * Relocate all entries in the GOT table.
         * Bump bss entries to _edata + dtb size
         */
1:        ldr    r1, [r11, #0]        @ relocate entries in the GOT
        add    r1, r1, r0        @ This fixes up C references
        cmp    r1, r2            @ if entry >= bss_start &&
        cmphs    r3, r1            @       bss_end > entry
        addhi    r1, r1, r5        @    entry += dtb size
        str    r1, [r11], #4        @ next entry
        cmp    r11, r12
        blo    1b

        /* bump our bss pointers too */
        add    r2, r2, r5
        add    r3, r3, r5

#else

        /*
         * Relocate entries in the GOT table.  We only relocate
         * the entries that are outside the (relocated) BSS region.
         */
1:        ldr    r1, [r11, #0]        @ relocate entries in the GOT
        cmp    r1, r2            @ entry < bss_start ||
        cmphs    r3, r1            @ _end < entry
        addlo    r1, r1, r0        @ table.  This fixes up the
        str    r1, [r11], #4        @ C references.
        cmp    r11, r12
        blo    1b
#endif

not_relocated:    mov    r0, #0
1:        str    r0, [r2], #4        @ clear bss
        str    r0, [r2], #4
        str    r0, [r2], #4
        str    r0, [r2], #4
        cmp    r2, r3
        blo    1b

        /*
         * Did we skip the cache setup earlier?
         * That is indicated by the LSB in r4.
         * Do it now if so.
         */
        tst    r4, #1
        bic    r4, r4, #1
        blne    cache_on

/*
* The C runtime environment should now be setup sufficiently.
* Set up some pointers, and start decompressing.
*   r4  = kernel execution address
*   r7  = architecture ID
*   r8  = atags pointer
*/
        mov    r0, r4                                       //r0=r4=0x40008000(phys)
        mov    r1, sp            @ malloc space above stack //r1=sp=0x4235aad8(phys) 即自解压程序栈顶地址
        add    r2, sp, #0x10000    @ 64k max                //r2=sp+64k=0x4236aad8(phys) 这个64k相当于从堆中分配的内存给解压程序使用 
        mov    r3, r7
        bl    decompress_kernel
        bl    cache_clean_flush
        bl    cache_off
        mov    r1, r7            @ restore architecture number
        mov    r2, r8            @ restore atags pointer

#ifdef CONFIG_ARM_VIRT_EXT
        mrs    r0, spsr        @ Get saved CPU boot mode
        and    r0, r0, #MODE_MASK
        cmp    r0, #HYP_MODE        @ if not booted in HYP mode...
        bne    __enter_kernel        @ boot kernel directly  //跳转至0x4000800 rch/arm/kernel/head.S stext段运行

        adr    r12, .L__hyp_reentry_vectors_offset
        ldr    r0, [r12]
        add    r0, r0, r12

        bl    __hyp_set_vectors
        __HVC(0)            @ otherwise bounce to hyp mode

        b    .            @ should never be reached

        .align    2
.L__hyp_reentry_vectors_offset:    .long    __hyp_reentry_vectors - .
#else
        b    __enter_kernel
#endif

        .align    2
        .type    LC0, #object
LC0:        .word    LC0            @ r1
        .word    __bss_start        @ r2
        .word    _end            @ r3
        .word    _edata            @ r6
        .word    input_data_end - 4    @ r10 (inflated size location)
        .word    _got_start        @ r11
        .word    _got_end        @ ip
        .word    .L_user_stack_end    @ sp
        .word    _end - restart + 16384 + 1024*1024
        .size    LC0, . - LC0

#ifdef CONFIG_ARCH_RPC
        .globl    params
params:        ldr    r0, =0x10000100        @ params_phys for RPC
        mov    pc, lr
        .ltorg
        .align
#endif
```

```
159 00000248 <LC0>:
160      248:▼      00000248 ▼      @r1
161      24c:▼      00359ab8 ▼      @r2 __bss_start        bss段起始链接地址  恰好等于压缩后的内核镜像大小，等于_edata
162      250:▼      00359ad4 ▼      @r3 _end               应该总的结束地址即bss段段结束链接地址
163      254:▼      00359ab8 ▼      @r6 _edata             数据段结束链接地址
164      258:▼      00359a7b ▼      @r10 input_data_end - 4 压缩前的内核镜像Image大小存放地址，小端格式存放
165      25c:▼      00359a8c ▼      @r11 _got_start
166      260:▼      00359ab4 ▼      @ip _got_end           
167      264:▼      0035aad8 ▼      @sp L_user_stack_end   自解压代码栈结束地址
168      268:▼      0045da0c ▼      @  _end - restart + 16384 + 1024*1024  restart链接地址为0xc0

编译后的zImage实际大小：3513016 ==>0x359ab8  arch/arm/boot/Image
二进制打开zImage：
0359a70: d34c fbff b0ff 005f 2281 e900 4092 0000
0x00924000=9584640 约9.4MB
编译后的Image实际大小：9584640  arch/arm/boot/Image
```

```
145 void
146 decompress_kernel(unsigned long output_start, unsigned long free_mem_ptr_p,
147 ▼       ▼       unsigned long free_mem_ptr_end_p,                                                                                  
148 ▼       ▼       int arch_id)
```

```
1351 __enter_kernel:
1352 ▼       ▼       mov▼    r0, #0▼ ▼       ▼       @ must be 0    //r0=0
1353  ARM(▼  ▼       mov▼    pc, r4▼ ▼       )▼      @ call kernel  //跳转至0x40008000运行
1354  M_CLASS(▼      add▼    r4, r4, #1▼     )▼      @ enter in Thumb mode for M class
1355  THUMB(▼▼       bx▼     r4▼     ▼       )▼      @ entry point is always ARM for A/R classes
```

总结：

1、通过从LC0中获取zImage相关的参数信息

2、将计算zImage与Image的位置关系，判断是否存在重叠，如重则将zImage充电位

3、将zImage解压出Image至指定地址

2、跳转至Image中运行

![.png](image/.png)
