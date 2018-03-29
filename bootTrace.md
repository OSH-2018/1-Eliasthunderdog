# LINUX系统启动过程分析

实验环境：fedora 27（4.15.10内核） qemu虚拟机

## 实验工具

gdb, linux kernel(最新 4.15.12稳定版）, qemu-system-x86-64

## 实验步骤

1. 在宿主机上编译内核，解压内核后在目录下运行

``` shell
make menuconfig
```

在linux kernel hacking 中选中debug info 生成带有调试信息的内核

然后运行

``` shell
make -j 4
make module_install
make install
```

进行编译，内核安装，再运行

``` shell
make bzImage
```

生成不带调试信息的内核，这个是给虚拟机使用的内核。

此时在linux内核目录下已经生成了vmlinux未经压缩的，带调试信息的内核，可以用GDB加载进行调试。

2. 宿主机上运行qemu，只需要内核和initramfs,并不需要安装操作系统即可进行调试

``` shell
sudo qemu-system-x86_64 debian9.img -kernel /usr/src/linux-4.15.12/arch/x86/boot/bzImage -gdb tcp::1234 -S -append "nokaslr" -nographic
```

其中nokaslr是禁用了一个随机映射函数入口地址的技术，不禁用的话无法在函数入口处设置breakpoint并成功停下。

3. 启动gdb开始调试

启动GDB，并载入vmlinux，再执行 target remote localhost:1234，即可连接到qemu开始调试。

## 实验结果

### 引导程序加载

0x7c00: 0x7c00是早期工程师设计的一个“魔数”，为的是每次都将引导程序加载到这里，从这里开始引导操作系统，
之所以放在这里是方便给***留出空间预计的占用，引导操作系统的时候系统还工作在实模式状态。

注： 这个在只给虚拟机指定内核和initramfs的状况下是没法追踪到这个事件的，在一个正常安装的虚拟机上调试才能捕捉到。

### 进入保护模式：go_to_protected_mode 函数

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

### 内核启动: start_kernel()函数

这个函数非常复杂，分为几个主要的阶段。

#### set_task_stack_end_magic()

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


#### trap_init() 函数

这个函数的作用是初始化陷阱(trap)，trap在隔离用户态和内核态中起到了重要的作用，运行在内核态的程序要调用内核提供的服务，就要通过trap，trap指令使控制权由用户应用程序转移到内核。