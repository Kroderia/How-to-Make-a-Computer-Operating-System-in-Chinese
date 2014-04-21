## 第3章: 使用GRUB进行初次启动

#### boot都干些什么？

一台x86计算机，从开机开始，它就走在了一条复杂的路上，直到控制权交给内核“主”程序（`kmain()`）。在这一章，我们只考虑BIOS启动，而不是其继任者UEFI。

BIOS的启动顺序是：识别RAM -> 硬件识别/初始化 -> 引导顺序（BootSequence）

对我们而言，最重要的一步是“引导顺序”。此时的BIOS刚完成初始化，尝试把控制权移交给下一步的bootloader。

在“引导顺序”中，BIOS会尝试找到一个“启动设备”（如软盘、硬盘、CD、usb闪存设备或是网络）。我们的操作系统会使用硬盘进行初次启动（但它也可以从CD或usb闪存设备启动）。一个“可启动”的设备是这样的：设备的启动扇区的第511和512字节内容分别为`0x55`和`0xAA`，这是有效标识符。

BIOS通过载入每个设备的前512个字节到物理内存中（以物理地址`0x7C00`开始的1KiB，32KiB的最后1KiB）, 来识别设备是否可启动。当识别到有效标识符，BIOS会通过一个跳转指令，把控制权移交给`0x7C00`内存地址，以执行启动扇区代码。

(译注：关于`0x7C00`的起源，可以参考[http://www.glamenv-septzen.net/en/view/6](http://www.glamenv-septzen.net/en/view/6))

在这一整个过程中，CPU是在16位实模式（Real Mode，x86 CPU的默认状态, 这是为了后向兼容）下运行的。为了在内核中执行32位指令，我们需要一个bootloader来使CPU切换到保护模式（Protexted Mode）。

#### 什么是GRUB?

> GNU GRUB(GNU GRand Unified Bootloader的缩写), 是GNU计划的一个bootloader包。GRUB是自由软件基金会关于多重引导规范的一个参考实现。它让用户可以在一台计算机安装的多个操作系统中选择一个，或是指定一个操作系统下可用的内核配置。

简而言之，GRUB是机器（bootloader）第一个启动的东西，它简化了载入硬盘中内核的过程。


#### 为什么使用GRUB?

* GRUB非常易用
* 不需要16位代码就能轻易地载入32位内核
* 提供Linux，Windows及其他系统的多重引导
* 容易载入内存中的额外模块

#### 怎么使用GRUB?

GRUB遵循多重引导规范，其可执行的二进制（程序）是32位的，而且必须在前8192字节内包含一个特定header（多重引导的header）。我们的内核会使用ELF可执行文件（"Executable and Linkable Format"，“可执行可链接格式”，是大部分UNIX系统中的可执行文件的通用标准文件格式）。

这个内核的首次引导顺序是用汇编来写的：[start.asm](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/start.asm)。还会用一个链接器文件来定义可执行结构：[linker.ld](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/linker.ld)。

这个启动过程同时会初始化一些C++的运行环境，下一章会详细介绍。

多重引导的header结构:

```
struct multiboot_info {
	u32 flags;
	u32 low_mem;
	u32 high_mem;
	u32 boot_device;
	u32 cmdline;
	u32 mods_count;
	u32 mods_addr;
	struct {
		u32 num;
		u32 size;
		u32 addr;
		u32 shndx;
	} elf_sec;
	unsigned long mmap_length;
	unsigned long mmap_addr;
	unsigned long drives_length;
	unsigned long drives_addr;
	unsigned long config_table;
	unsigned long boot_loader_name;
	unsigned long apm_table;
	unsigned long vbe_control_info;
	unsigned long vbe_mode_info;
	unsigned long vbe_mode;
	unsigned long vbe_interface_seg;
	unsigned long vbe_interface_off;
	unsigned long vbe_interface_len;
};
```

可以用```mbchk kernel.elf```命令来检查kernel.elf文件是否符合多重引导标准。同样可以用```nm -n kernel.elf```来检查在ELF二进制中的对象的偏移。

#### 为内核与grub创建一个磁盘镜像

这个[diskimage.sh](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/sdk/diskimage.sh)脚本可以生成一个QEMU可用的硬盘镜像。

第一步，使用qemu-img创建一个硬盘镜像（c.img）：

```
qemu-img create c.img 2M
```

现在用fdisk进行磁盘分区

```
fdisk ./c.img

# Switch to Expert commands
> x

# Change number of cylinders (1-1048576)
> c
> 4

# Change number of heads (1-256, default 16):
> h
> 16

# Change number of sectors/track (1-63, default 63)
> s
> 63

# Return to main menu
> r

# Add a new partition
> n

# Choose primary partition
> p

# Choose partition number
> 1

# Choose first cylinder (1-4, default 1)
> 1

# Choose last cylinder, +cylinders or +size{K,M,G} (1-4, default 4)
> 4

# Toggle bootable flag
> a

# Choose first partition for bootable flag
> 1

# Write table to disk and exit
> w
```

现在需要使用losetup，把刚创建的分区绑定到回环设备（loop-device，可以让文件通过块设备的方式访问）。通过**偏移量 = 开始扇区 * 每扇区字节数** 计算分区偏移量，作为参数传递。

```fdisk -l -u c.img``` 得到 63 * 512 = 32256.

（译注：以下命令请使用sudo执行）

```
losetup -o 32256 /dev/loop1 ./c.img
```

在新设备上创建EXT2文件系统

```
mke2fs /dev/loop1
```

复制文件到已挂载磁盘上

```
mount  /dev/loop1 /mnt/
cp -R bootdisk/* /mnt/
umount /mnt/
```

在磁盘上安装GRUB

```
grub --device-map=/dev/null << EOF
device (hd0) ./c.img
geometry (hd0) 4 16 63
root (hd0,0)
setup (hd0)
quit
EOF
```

最后脱离回环设备

```
losetup -d /dev/loop1
```

####参见

* [GNU GRUB on Wikipedia](http://en.wikipedia.org/wiki/GNU_GRUB)
* [Multiboot specification](https://www.gnu.org/software/grub/manual/multiboot/multiboot.html)