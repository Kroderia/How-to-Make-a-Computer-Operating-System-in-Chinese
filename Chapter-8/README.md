## 第八章: 内存管理: 物理内存和虚拟内存

在关于GDT的一章，我们可以看到，使用segmentation，一个物理内存地址可以基于segment selector 和 offset 计算得来。

在这一章，我们会实现paging（分页），paging可以将segmentation中线性地址转换成真实的物理地址。

#### 为什么我们需要 paging?

Paging 的使用能帮助我们的内核实现如下功能:

* 使hard-drive(硬盘)可以作为内存，并且不会被真实的RAM内存大小所限制
* 使每一个进程都有自己专属的内存空间
* to allow and unallow memory space in a dynamic way


在一个分页的系统中，每一个进程都可以在自己的4GB内存运行，同时不会影响其他进程以及内核的内存空间。分页机简化了multitasking。

![Processes memories](./processes.png)

#### 如何工作?

线性地址到物理地址的转换通过以下几步完成:

1. 处理器使用寄存器'CR3'获取 pages directory 的物理地址
2. 线性地址的前10 bits代表pages directory的offset(偏移），从0到1023，指向pages directory 的一个entry。这个entry包含着pages table的物理地址。
3. 线性地址中接下来的10 bits代表pages table中的offset，指向pages table 中的entry。这个entry指向一个4ko 的page
4. 线性地址的最后12 bits代表在4ko page中的offset(从0到4095), 指向4ko page中的一个位置。

![Address translation](./paging_memory.png)

#### Pages table and directory的格式

The two types of entries (table and directory) look like the same. Only the field in gray will be used in our OS.
这两种形式的entries (table and directory)看起来很像。 我们的OS只用到了下图灰色部分。

![Page directory entry](./page_directory_entry.png)

![Page table entry](./page_table_entry.png)

* `P`: 指明这个 page 或者 table 是否存在于物理内存
* `R/W`: 指明这个 page 或者 table 是否可写 (1代表可写)
* `U/S`: 为1时允许 non-preferred tasks的访问
* `A`: 指明这个 page 或者 table 是否可以访问
* `D`: (only for pages table) 指明这个page是否被写，dirty bit
* `PS` (only for pages directory) 指明page的大小:
    * 0 = 4ko
    * 1 = 4mo

**注意:** Pages directory 或者 pages table中的物理地址只保存前20 bits, 这是因为这些地址是4ko对齐的，因此最后的12 bits 为0.
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

但是在此之前，我们需要初始化我们的pages directory，这个pages directory至少包含一个pages table。



