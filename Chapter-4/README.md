## 第4章: OS的骨干与C++运行环境


#### C++ kernel run-time

一个用C++编成的内核，这跟用C制作一个内核很相似，不过其中有些必须重视的误区（运行环境支持，构造器等...）

编译器会默认所有C++运行时支持是可用的。不过由于我们不会把libsupc++嵌入C++内核中，因此需要先添加一些基本的函数。它们都被包含在[cxx.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/cxx.cc)文件中。


**注意：** 在虚拟内存和内存分页没有被初始化之前，`new`和`delete`都不能使用。


#### 基本C/C++函数

内核代码不能使用标准程序库。因此需要添加一些基本函数来管理内存和string。


```cpp
void 	itoa(char *buf, unsigned long int n, int base);

void *	memset(char *dst,char src, int n);
void *	memcpy(char *dst, char *src, int n);

int 	strlen(char *s);
int 	strcmp(const char *dst, char *src);
int 	strcpy(char *dst,const char *src);
void 	strcat(void *dest,const void *src);
char *	strncpy(char *destString, const char *sourceString,int maxLength);
int 	strncmp( const char* s1, const char* s2, int c );
```


这些函数被定义在[string.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/string.cc), [memory.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/memory.cc), [itoa.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/itoa.cc)

#### C types

下面需要定义一些变量类型。主要用到的是unsigned类型（使用所有比特存储整数。而signed类型的有一位用于标记正负）：


```cpp

typedef unsigned char 	u8;
typedef unsigned short 	u16;
typedef unsigned int 	u32;
typedef unsigned long long 	u64;

typedef signed char 	s8;
typedef signed short 	s16;
typedef signed int 		s32;
typedef signed long long	s64;
```

#### 编译内核

编译内核不同于编译一个Linux可执行文件。我们不能使用标准程序库，而且必须对系统没有依赖。

这个[Makefile](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/Makefile)定义了如何编译并链接内核。

在x86架构中，gcc/g++/ld需要以下参数：


```
# Linker
LD=ld
LDFLAG= -melf_i386 -static  -L ./  -T ./arch/$(ARCH)/linker.ld

# C++ compiler
SC=g++
FLAG= $(INCDIR) -g -O2 -w -trigraphs -fno-builtin  -fno-exceptions -fno-stack-protector -O0 -m32  -fno-rtti -nostdlib -nodefaultlibs 

# Assembly compiler
ASM=nasm
ASMFLAG=-f elf -o
```