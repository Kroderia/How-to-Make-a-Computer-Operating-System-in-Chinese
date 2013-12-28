如何制作一个操作系统
=======================================

**注意**: 这个repo是我以前的课程的重制. [它是我还在高校时最早的一个project.](https://github.com/SamyPesse/devos). 我还在重构其中一些部分. 原课程是法语教授的, 而且我也不是英语母语的. 我在空余时间继续改进它.

**源码**: 所有系统源码都放在`src`文件夹内. 每一步会包含其相关文件的链接.

**投稿**: 这个课程是开放投稿的. 如果发现错误, 随时通过issue或者直接pull request来改正它.

**问题**: 同样可以随意通过issue来问问题. 请不要给我发邮件.

你可以在Twitter[@SamyPesse](https://twitter.com/SamyPesse)上关注我, 在[Flattr](https://flattr.com/profile/samy.pesse)或[Gittip](https://www.gittip.com/SamyPesse/)上支持我.

### 我们在制作怎样的操作系统?

目标是使用C++构建一个基于UNIX, 非常简单的操作系统, 而不只是一个"概念验证". 这个操作系统可以启动, 运行一个userland shell, 可以扩展.



[![Screen](https://raw.github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/master/preview.png)](https://raw.github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/master/preview.png)

### 摘要

#### [第1章: x86与本操作系统简介](Chapter-1/README.md)

#### [第2章: 建立开发环境](Chapter-2/README.md)

#### [第3章: 使用GRUB进行初次启动](Chapter-3/README.md)

#### [第4章: OS的中枢与C++运行环境](Chapter-4/README.md)

#### [第5章: 管理x86架构的基类](Chapter-5/README.md)

#### Chapter 6: GDT

#### Chapter 7: IDT and interrupts

#### Chapter 8: Memory management: physical and virtual

#### Chapter 9: Process management and multitasking

#### Chapter 10: External program execution: ELF files

#### Chapter 11: Userland and syscalls

#### Chapter 12: Modular drivers

#### Chapter 13: Some basics modules: console, keyboard

#### Chapter 14: IDE Hard disks

#### Chapter 15: DOS Partitions

#### Chapter 16: EXT2 read-only filesystems

#### Chapter 17: Standard C library (libC)

#### Chapter 18: UNIX basic tools: sh, cat

#### Chapter 19: Lua interpreter

