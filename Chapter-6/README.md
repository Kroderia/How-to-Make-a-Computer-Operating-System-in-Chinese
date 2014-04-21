## 第六章 GDT

首先感谢GRUB,如此一来我们的内核不再是在实模式，而是已经在保护模式，这个模式允许我们来使用微处理器的所有的可能性，包括虚拟内存管理、分页以及安全的多任务。

#### 到底什么是 GDT？

GDT（“Global Descriptor Table”，全局描述符表）是一种数据结构，通常用于定义不同的内存区域：基地址、大小以及访问权限，如执行和写入。这些内存区域我们将其称之为“段”。

我们下面将用GDT来定义不同的内存段：

* *"code"*：内核代码，用来存储可执行文件二进制代码
* *"data"*：内核数据
* *"stack"*：内存栈，用来存储内核执行期间的调用栈
* *"ucode"*：用户代码，用来存储用户程序的可执行二进制代码
* *"udata"*：用户程序数据
* *"ustack"*：用户栈，用来存储userland中运行程序的调用栈

#### 如何来加载GDT？

GRUB会来初始化GDT，但是这个GDT与我们的内核不符。GDT是使用LGDT这一汇编指令加载的。GDT描述的结构位置如下：

![GDTR](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/raw/master/Chapter-6/gdtr.png)

C结构如下：

```cpp
struct gdtr {
    u16 limite;
    u32 base;
} __attribute__ ((packed));
```

**注意：** ```__attribute__ ((packed))```意味着告知gcc，该结构应使用尽量少的内存。倘若没有它，gcc会额外包含一些字节，在运行中对内存对齐及访问进行优化。

现在需要定义我们的GDT，并用LGDT载入。GDT可存储于内存中的任意位置。其地址存储于GDTR寄存器，以告知进程。

GDT表是由下面的结构组成：

![GDTR](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/raw/master/Chapter-6/gdtentry.png)

C结构：

```cpp
struct gdtdesc {
    u16 lim0_15;
    u16 base0_15;
    u8 base16_23;
    u8 acces;
    u8 lim16_19:4;
    u8 other:4;
    u8 base24_31;
} __attribute__ ((packed));
```

#### 如何定义我们的GDT表？
现在需要定义我们的GDT表，并使用GDTR寄存器来加载它。

我们将GDT存储在下面的地址：

```cpp
#define GDTBASE 0x00000800
```

函数**init_gdt_desc**[x86.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/x86.cc)用于初始化一个GDT段描述器。


```cpp
void init_gdt_desc(u32 base, u32 limite, u8 acces, u8 other, struct gdtdesc *desc)
{
    desc->lim0_15 = (limite & 0xffff);
    desc->base0_15 = (base & 0xffff);
    desc->base16_23 = (base & 0xff0000) >> 16;
    desc->acces = acces;
    desc->lim16_19 = (limite & 0xf0000) >> 16;
    desc->other = (other & 0xf);
    desc->base24_31 = (base & 0xff000000) >> 24;
    return;
}
```

函数**init_gdt**会初始化GDT，函数的一些部分，我们会在以后进行解释，它们也会在多任务中被使用。

```cpp
void init_gdt(void)
{
    default_tss.debug_flag = 0x00;
    default_tss.io_map = 0x00;
    default_tss.esp0 = 0x1FFF0;
    default_tss.ss0 = 0x18;

    /* initialize gdt segments */
    init_gdt_desc(0x0, 0x0, 0x0, 0x0, &kgdt[0]);
    init_gdt_desc(0x0, 0xFFFFF, 0x9B, 0x0D, &kgdt[1]);  /* code */
    init_gdt_desc(0x0, 0xFFFFF, 0x93, 0x0D, &kgdt[2]);  /* data */
    init_gdt_desc(0x0, 0x0, 0x97, 0x0D, &kgdt[3]);      /* stack */

    init_gdt_desc(0x0, 0xFFFFF, 0xFF, 0x0D, &kgdt[4]);  /* ucode */
    init_gdt_desc(0x0, 0xFFFFF, 0xF3, 0x0D, &kgdt[5]);  /* udata */
    init_gdt_desc(0x0, 0x0, 0xF7, 0x0D, &kgdt[6]);      /* ustack */

    init_gdt_desc((u32) & default_tss, 0x67, 0xE9, 0x00, &kgdt[7]); /* descripteur de tss */

    /* initialize the gdtr structure */
    kgdtr.limite = GDTSIZE * 8;
    kgdtr.base = GDTBASE;

    /* copy the gdtr to its memory area */
    memcpy((char *) kgdtr.base, (char *) kgdt, kgdtr.limite);

    /* load the gdtr registry */
    asm("lgdtl (kgdtr)");

    /* initiliaz the segments */
    asm("   movw $0x10, %ax \n \
    movw %ax, %ds   \n \
    movw %ax, %es   \n \
    movw %ax, %fs  \n \
    movw %ax, %gs   \n \
    ljmp $0x08, $next   \n \
    next:       \n");
}
```
