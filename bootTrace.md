# LINUX系统启动过程分析

## 引导程序加载

0x7c00: 0x7c00是早期工程师设计的一个“魔数”，为的是每次都将引导程序加载到这里，从这里开始引导操作系统，
之所以放在这里是方便给***留出空间预计的占用，引导操作系统的时候系统还工作在实模式状态。

## 进入保护模式：go_to_protected_mode 函数

在 arch/x86/boot/pm.c中定义了go_to_protected_mode函数，将操作系统的模式从实模式转向保护模式

进行的主要操作：

1. real_mode_switch_hook

嗯这个还没弄清楚干什么用的。

2. 打开A20 enable_a20

打开A20地址线，A20是为了让80286向前兼容设置的。

3. mask_all_interrupts

在实模式下的中断已经不用了，所以要全部mask掉，直接对内存进行位操作即可。

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

载入 idt，即中断描述符表，每一个中断向量在表中都有相应的处理程序的入口地址。

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

        static struct gdt_ptr gdt;

        gdt.len = sizeof(boot_gdt)-1;
        gdt.ptr = (u32)&boot_gdt + (ds() << 4);

        asm volatile("lgdtl %0" : : "m" (gdt));
        // 最重要的是这一条，用内置的lgdtl指令载入gdt
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

其中STACK_END_MAGIC也是“魔数”，是设计者根据估计预先定义好的数。


### trap_init() 函数

这个函数的作用是初始化陷阱(trap)，trap在隔离用户态和内核态中起到了重要的作用，运行在内核态的程序要调用内核提供的服务，就要通过trap，trap指令使控制权由用户应用程序转移到内核。