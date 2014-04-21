## 第7章: IDT与中断

中断（interrupt）是由硬件或软件产生的信号，表示发生了急需处理的事件。

中断有3种：

- **硬件中断(Hardware interrupts)：** 由外部设备（如键盘，鼠标，硬盘...）向处理器发送的中断。硬件中断是一种在轮询循环中等待外部事件，减少处理器的宝贵时间的方式。

- **软件中断(Software interrupts)：** 由软件自发发起的。用于管理系统调用。

- **异常(Exceptions)：** 用于程序中出现超出其所能处理的错误或事件（如除零、页错误...）


#### 键盘样例：

当用户敲击键盘上的一个键， 键盘控制器会发送一个中断信号到中断控制器（Interrupt Controller）。如果它没有被屏蔽，控制器会为它给处理器发送信号。处理器会执行一套流程来管理中断（键的按击或松开）。这样的一套流程可以完成如接收按键并输出到屏幕的过程。当这个字符处理过程完成后，原来被中断的任务会被回复。


#### PIC是什么？

[PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller)（Programmable interrupt controller，可编程中断控制器）被用于合并多种来源的中断到一个或多个CPU管线（lines），同时为其中断输出分配优先级。当设备有多个中断输出发生时，它使它们遵循优先级来发生。

最有名的PIC是8259A。每个8259A可以处理8个设备，但是大部分计算机有多个控制器：一个主控制器（master）和一个从控制器（slave），它使计算机可以管理来自14个设备的中断。

在本章中，我们需要为这个控制器编程，使其初始化，然后屏蔽中断。

#### IDT是什么？

> 中断描述符表（IDT）是x86架构用于实现中断向量表的数据结构。IDT被处理器用于决定对中断及异常作出何种应答。

我们的内核将使用IDT来定义中断发生时调用的不同程序。

与GDT类似，IDT使用LIDTL汇编指令来加载。一个IDT的位置结构如下描述：

```cpp
struct idtr {
	u16 limite;
	u32 base;
} __attribute__ ((packed));
```

IDT表由如下结构的IDT段所构成：

```cpp
struct idtdesc {
	u16 offset0_15;
	u16 select;
	u16 type;
	u16 offset16_31;
} __attribute__ ((packed));
```

**注意：** ```__attribute__ ((packed))``` 请参考上一章。

现在需要定义我们的IDT表并使用LIDTL来载入它。IDT表可以存储在内存中的任何位置，其地址存储与IDTR寄存器以告知进程。

以下是一个通用的中断表（Maskable hardware interrupt，IRQ，可屏蔽硬件信号）：


| IRQ   |         Description        |
|:-----:| -------------------------- |
| 0 | Programmable Interrupt Timer Interrupt | 
| 1 | Keyboard Interrupt | 
| 2 | Cascade (used internally by the two PICs. never raised) | 
| 3 | COM2 (if enabled) | 
| 4 | COM1 (if enabled) | 
| 5 | LPT2 (if enabled) | 
| 6 | Floppy Disk | 
| 7 | LPT1 | 
| 8 | CMOS real-time clock (if enabled) | 
| 9 | Free for peripherals / legacy SCSI / NIC | 
| 10 | Free for peripherals / SCSI / NIC | 
| 11 | Free for peripherals / SCSI / NIC | 
| 12 | PS2 Mouse | 
| 13 | FPU / Coprocessor / Inter-processor | 
| 14 | Primary ATA Hard Disk | 
| 15 | Secondary ATA Hard Disk |

#### 如何初始化中断？

以下是定义IDT段的一个简单方法

```cpp
void init_idt_desc(u16 select, u32 offset, u16 type, struct idtdesc *desc)
{
	desc->offset0_15 = (offset & 0xffff);
	desc->select = select;
	desc->type = type;
	desc->offset16_31 = (offset & 0xffff0000) >> 16;
	return;
}
```

然后初始化中断：

```cpp
#define IDTBASE	0x00000000
#define IDTSIZE 0xFF
idtr kidtr;
```


```cpp
void init_idt(void)
{
	/* Init irq */
	int i;
	for (i = 0; i < IDTSIZE; i++) 
		init_idt_desc(0x08, (u32)_asm_schedule, INTGATE, &kidt[i]); // 
	
	/* Vectors  0 -> 31 are for exceptions */
	init_idt_desc(0x08, (u32) _asm_exc_GP, INTGATE, &kidt[13]);		/* #GP */
	init_idt_desc(0x08, (u32) _asm_exc_PF, INTGATE, &kidt[14]);     /* #PF */
	
	init_idt_desc(0x08, (u32) _asm_schedule, INTGATE, &kidt[32]);
	init_idt_desc(0x08, (u32) _asm_int_1, INTGATE, &kidt[33]);
	
	init_idt_desc(0x08, (u32) _asm_syscalls, TRAPGATE, &kidt[48]);
	init_idt_desc(0x08, (u32) _asm_syscalls, TRAPGATE, &kidt[128]); //48
	
	kidtr.limite = IDTSIZE * 8;
	kidtr.base = IDTBASE;
	
	
	/* Copy the IDT to the memory */
	memcpy((char *) kidtr.base, (char *) kidt, kidtr.limite);

	/* Load the IDTR registry */
	asm("lidtl (kidtr)");
}
```

在IDT初始化后，需要配置PIC来激活中断。下面的这个函数会通过处理器的输出端口（port），写入两个PIC的内部寄存器来进行配置。


```io.outb``` 我们使用以下端口来配置PIC:

* Master PIC: 0x20 and 0x21
* Slave PIC: 0xA0 and 0xA1

PIC有两种寄存器：

* ICW (Initialization Command Word，初始化命令字): 重新初始化控制器
* OCW (Operation Control Word，操作命令字): 当初始化后配置控制器（用于屏蔽/取消屏蔽中断）

```cpp
void init_pic(void)
{
	/* Initialization of ICW1 */
	io.outb(0x20, 0x11);
	io.outb(0xA0, 0x11);

	/* Initialization of ICW2 */
	io.outb(0x21, 0x20);	/* start vector = 32 */
	io.outb(0xA1, 0x70);	/* start vector = 96 */

	/* Initialization of ICW3 */
	io.outb(0x21, 0x04);
	io.outb(0xA1, 0x02);

	/* Initialization of ICW4 */
	io.outb(0x21, 0x01);
	io.outb(0xA1, 0x01);

	/* mask interrupts */
	io.outb(0x21, 0x0);
	io.outb(0xA1, 0x0);
}
```

#### PIC ICW配置细节

寄存器必须按顺序配置。

**ICW1 (port 0x20 / port 0xA0)**
```
|0|0|0|1|x|0|x|x|
         |   | +--- with ICW4 (1) or without (0)
         |   +----- one controller (1), or cascade (0)
         +--------- triggering by level (level) (1) or by edge (edge) (0)
```

**ICW2 (port 0x21 / port 0xA1)**
```
|x|x|x|x|x|0|0|0|  
 | | | | |
 +----------------- base address for interrupts vectors
```

**ICW2 (port 0x21 / port 0xA1)**

主控制器：

```
|x|x|x|x|x|x|x|x|
 | | | | | | | |
 +------------------ slave controller connected to the port yes (1), or no (0)
```

从控制器：

```
|0|0|0|0|0|x|x|x|  pour l'esclave
           | | |
           +-------- Slave ID which is equal to the master port
```

**ICW4 (port 0x21 / port 0xA1)**

用于定义控制器可运行在何种模式。

```
|0|0|0|x|x|x|x|1|
       | | | +------ mode "automatic end of interrupt" AEOI (1)
       | | +-------- mode buffered slave (0) or master (1)
       | +---------- mode buffered (1)
       +------------ mode "fully nested" (1)
```

#### 为什么IDT段的offset是ASM函数？

你应该留意到在初始化IDT段时，我对段使用了汇编编写的offset。The se different functions are defined in [x86int.asm](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/x86int.asm) and are following the scheme:

```
%macro	SAVE_REGS 0
	pushad 
	push ds
	push es
	push fs
	push gs 
	push ebx
	mov bx,0x10
	mov ds,bx
	pop ebx
%endmacro

%macro	RESTORE_REGS 0
	pop gs
	pop fs
	pop es
	pop ds
	popad
%endmacro

%macro	INTERRUPT 1
global _asm_int_%1
_asm_int_%1:
	SAVE_REGS
	push %1
	call isr_default_int
	pop eax	;;a enlever sinon
	mov al,0x20
	out 0x20,al
	RESTORE_REGS
	iret
%endmacro
```

这些宏用于定义防止不同寄存器损坏的中断段。它在多任务中非常有用。

（原文：These macros will be used to define interrupt segment that will prevent corruption of the different registries）
