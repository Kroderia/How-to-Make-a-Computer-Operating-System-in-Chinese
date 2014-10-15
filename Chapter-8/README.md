## 第八章: 内存管理: 物理内存和虚拟内存

在关于GDT的一章，我们可以看到，使用segmentation(分段)，一个物理内存地址可以基于segment selector 和 offset 计算得来。

在这一章，我们会实现paging(分页)，paging可以将segmentation中线性地址转换成真实的物理地址。

#### 为什么我们需要 paging?

Paging 的使用能帮助内核实现如下功能:

* 使hard-drive(硬盘)可以成为内存，且不被真实的RAM内存大小所限制
* 使每一个进程都有自己专属的内存空间
* to allow and unallow memory space in a dynamic way


在一个分页的系统中，每一个进程都可以在自己的完整4GB内存运行，同时不影响其他进程以及内核的内存空间。分页机简化了multitasking。

![Processes memories](https://raw.githubusercontent.com/SamyPesse/How-to-Make-a-Computer-Operating-System/master/Chapter-8/processes.png)

#### 如何工作?

线性地址到物理地址的转换通过以下几步完成:

1. 处理器使用寄存器 'CR3' 获取 pages directory 的物理地址
2. 线性地址的前10 bits代表pages directory的offset(偏移)，从0到1023，指向pages directory 的一个entry。这个entry包含着pages table的物理地址。
3. 线性地址中接下来的10 bits代表pages table中的offset，指向pages table 中的entry。这个entry指向一个4ko 的page
4. 线性地址的最后12 bits代表在4ko page中的offset(从0到4095), 指向4ko page中的一个位置。

![Address translation](https://raw.githubusercontent.com/SamyPesse/How-to-Make-a-Computer-Operating-System/master/Chapter-8/paging_memory.png)

#### Pages table and directory的格式

这两种形式的entries (table and directory)看起来很像。 在我们的OS中，只用到了灰色部分。

![Page directory entry](https://raw.githubusercontent.com/SamyPesse/How-to-Make-a-Computer-Operating-System/master/Chapter-8/page_directory_entry.png)

![Page table entry](https://raw.githubusercontent.com/SamyPesse/How-to-Make-a-Computer-Operating-System/master/Chapter-8/page_table_entry.png)

* `P`: 指明这个 page 或者 table 是否存在于物理内存
* `R/W`: 指明这个 page 或者 table 是否可写 (1代表可写)
* `U/S`: 为1时允许 non-preferred tasks的访问
* `A`: 指明这个 page 或者 table 是否可以访问
* `D`: (only for pages table) 指明这个page是否被写，dirty bit
* `PS` (only for pages directory) 指明page的大小:
    * 0 = 4ko
    * 1 = 4mo

**注意:** Pages directory 或者 pages table中的物理地址只保存前20 bits, 因为这些地址是4ko对齐的，所以最后的12 bits 为0.
* 一个 pages directory 或者 pages table 占用 1024*4 = 4096 bytes = 4k
* 一个 pages table 可以寻址 1024 * 4k = 4 Mo
* 一个 pages directory 可以寻址 1024 * (1024 * 4k) = 4 Go

#### 如何使能pagination(分页)?

我们只需要设置 'CR0' 寄存器的 bit 31 为1 即可:

```asm
asm("  mov %%cr0, %%eax; \
       or %1, %%eax;     \
       mov %%eax, %%cr0" \
       :: "i"(0x80000000));
```

在此之前，需要初始化我们的pages directory，这个pages directory至少包含一个pages table。

**译者注:**

1. 本章提到的linear address(线性地址) 也经常被称之为virtual address(虚拟地址)
2. Ko, Mo, Go 中的o代表octet, 1 octet = 8 bits。

