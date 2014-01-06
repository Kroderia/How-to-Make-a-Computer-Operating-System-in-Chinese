## 第六章 GDT

首先感谢 GRUB，如此一来我们的内核不再是在实模式，而是已经在保护模式，这个模式允许我们来使用微处理器的所有的可能性，包括虚拟内存管理，分页，以及安全的多任务。

#### 到底什么是 GDT？

GDT（"Global Descriptor Table）是一种数据结构，通常来用作给不同的内存区域做定义：包括基地址，大小，以及类似执行或者写这样的权限。这些内存区域我们将其称之为“段”。

我们下面将用 GDT 来给不同的内存区段做定义：

* *"code"*： 内核代码，用来存储可执行文件二进制代码
* *"data"*： 内核数据
* *"内核栈"*：用来存储内核执行期间的调用栈
* *"用户代码"*：用户代码，用来存储用户程序的可执行二进制代码
* *"用户程序数据"*：用户程序数据
* *"用户栈"*：用来存储用户程序执行期间的调用栈

#### 如何来加载我们 GDT？

GRUB 会来初始化 GDT，但是这个 GDT 并不会与我们的内核相对应。
我们会使用 LGDT 汇编指令来加载 GDT。GDT 描述的结构位置如下：

![GDTR](./gdtr.png)

C 语言结构如下：

```cpp
struct gdtr {
    u16 limite;
    u32 base;
} __attribute__ ((packed));
```

**注意：** ```__attribute__ ((packed))``` 意味着告诉 gcc 该结构应该用尽量少的内存。倘若没有它，gcc 就会引入一些字节来对内存分配进行优化。
现在我们来对 GDT 表做个定义，以及然后来使用 LGDT。GDT 表是可以被存放在内存中任意我们想存的位置，它的地址应该告诉使用 GDTR 寄存器的进程。

我们的 GDT 表是由下面的结构来组成：

![GDTR](./gdtentry.png)

然后是 C 结构：

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

#### 如果来定义我们的 GDT 表？
我们现在需要来给我们的 GDT 表做一个定义，并且最终使用 GDTR 寄存器来加载它。

我们将 GDT 存储在下面的地址：

```cpp
#define GDTBASE 0x00000800
```

函数 **init_gdt_desc** [x86.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/x86.cc) 初始化一个 GDT 段描述器。


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

并且函数 **init_gdt**  会初始化 GDT，我们稍后会解释这个函数的部分，并且会在多任务中使用它。

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

<table><tr><td><a href="../Chapter-5/README.md" >&larr; Previous</a></td><td><a href="../Chapter-7/README.md" >Next &rarr;</a></td></tr></table>
