# LINUX系统启动过程分析

## 引导程序加载

0xx7c00:

## 进入保护模式：go_to_protected_mode

在 arch/x86/boot/pm.c中定义了go_to_protected_mode函数，将操作系统的模式从实模式转向保护模式

进行的主要操作：

1. real_mode_switch_hook



2. 打开A20 enable_a20

打开A20地址线，A20是为了让80286向前兼容设置的。

3. mask_all_interrupts

在实模式下的中断已经不用了，所以要全部mask掉

``` c
static void mask_all_interrupts(void)
{
        outb(0xff, 0xa1);       /* Mask all interrupts on the secondary PIC */
        io_delay();
        outb(0xfb, 0x21);       /* Mask all but cascade on the primary PIC */
        io_delay();
}
```

4. setup_idt

载入 idt

``` c
static void setup_idt(void)
{
        static const struct gdt_ptr null_idt = {0, 0};
        asm volatile("lidtl %0" : : "m" (null_idt));
}
```

5. setup_gdt

载入GDT，这是保护模式分页机制的实现基础

``` c
static void setup_gdt(void)
{
        /* There are machines which are known to not boot with the GDT
           being 8-byte unaligned.  Intel recommends 16 byte alignment. */
        static const u64 boot_gdt[] __attribute__((aligned(16))) = {
                /* CS: code, read/execute, 4 GB, base 0 */
                [GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
                /* DS: data, read/write, 4 GB, base 0 */
                [GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
                /* TSS: 32-bit tss, 104 bytes, base 4096 */
                /* We only have a TSS here to keep Intel VT happy;
                   we don't actually use it for anything. */
                [GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
        };
        /* Xen HVM incorrectly stores a pointer to the gdt_ptr, instead
           of the gdt_ptr contents.  Thus, make it static so it will
           stay in memory, at least long enough that we switch to the
           proper kernel GDT. */
        static struct gdt_ptr gdt;

        gdt.len = sizeof(boot_gdt)-1;
        gdt.ptr = (u32)&boot_gdt + (ds() << 4);

        asm volatile("lgdtl %0" : : "m" (gdt));
}
```

## 内核启动: start_kernel()函数

这个函数非常复杂，分为几个主要的阶段。

### set_task_stack_end_magic()

这个函数是start_kernel函数调用的第一个函数，作用是设置栈的溢出标志

``` c

void set_task_stack_end_magic(struct task_struct *tsk)
{
    unsigned long *stackend;

    stackend = end_of_stack(tsk);
    *stackend = STACK_END_MAGIC;    /* for overflow detection */
}

```

其中STACK_END_MAGIC是所谓的“魔数”，是设计者根据估计预先定义好的数。

（这个函数具体有什么作用，为什么要设置这个）

### trap_init() 函数

这个函数的作用是初始化陷阱(trap)，trap在隔离用户态和内核态中起到了重要的作用，运行在内核态的程序要调用内核提供的服务，就要通过trap，trap指令使控制权由用户应用程序转移到内核。

