## Chapter 5: 管理x86架构的基类

我们已经知道如何编译我们的C++内核以及通过GRUB启动二进制程序了，后面可以用C/C++来做一些有趣的东西。

#### 在屏幕console中进行输出

我们会使用VGA默认模式(`03h`)向用户显示文本。通过显存的`0xB800`地址就可以直接访问屏幕了。屏幕分辨率为80x25，每个字符用2字节来定义：一个是字符码，一个是格式标识符。这意味着这个显存的大小为4000B（80 x 25 x 2B）。

IO类（[io.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc)）：
* **x,y**：定义指针在屏幕中的位置
* **real_screen**：定义显存指针
* **putc(char c)**：在屏幕上显示一个字符，以及管理指针位置
* **printf(char* s, ...)**：输出一个字符串

以下是[IO Class](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc)中的**putc**方法，用于在屏幕上显示一个字符，并更新(x, y)位置。

```cpp
/* put a byte on screen */
void Io::putc(char c){
	kattr = 0x07;
	unsigned char *video;
	video = (unsigned char *) (real_screen+ 2 * x + 160 * y);
	// newline
	if (c == '\n') {			
		x = 0;
		y++;
	// back space
	} else if (c == '\b') {	
		if (x) {
			*(video + 1) = 0x0;
			x--;
		}
	// horizontal tab
	} else if (c == '\t') {	
		x = x + 8 - (x % 8);
	// carriage return
	} else if (c == '\r') {	
		x = 0;
	} else {		
		*video = c;
		*(video + 1) = kattr;

		x++;
		if (x > 79) {
			x = 0;
			y++;
		}
	}
	if (y > 24)
		scrollup(y - 24);
}
```

还有一个知名且好用的方法：[printf](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc#L155)

```cpp
/* put a string in screen */
void Io::print(const char *s, ...){
	va_list ap;

	char buf[16];
	int i, j, size, buflen, neg;

	unsigned char c;
	int ival;
	unsigned int uival;

	va_start(ap, s);

	while ((c = *s++)) {
		size = 0;
		neg = 0;

		if (c == 0)
			break;
		else if (c == '%') {
			c = *s++;
			if (c >= '0' && c <= '9') {
				size = c - '0';
				c = *s++;
			}

			if (c == 'd') {
				ival = va_arg(ap, int);
				if (ival < 0) {
					uival = 0 - ival;
					neg++;
				} else
					uival = ival;
				itoa(buf, uival, 10);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				if (neg)
					print("-%s", buf);
				else
					print(buf);
			}
			 else if (c == 'u') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 10);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print(buf);
			} else if (c == 'x' || c == 'X') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 16);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print("0x%s", buf);
			} else if (c == 'p') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 16);
				size = 8;

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print("0x%s", buf);
			} else if (c == 's') {
				print((char *) va_arg(ap, int));
			} 
		} else
			putc(c);
	}

	return;
}
```

#### 汇编接口

有很多在汇编下可用的指令，在C下没有等价可用的实现（如cli、sti、in和out），因此我们需要为这些指令提供一个接口。

在C中，我们可以使用`asm()`指令来包含汇编语句，gcc会调用gas来编译它们。

**注意：** gas使用AT&T语法。

```cpp
/* output byte */
void Io::outb(u32 ad, u8 v){
	asmv("outb %%al, %%dx" :: "d" (ad), "a" (v));;
}
/* output word */
void Io::outw(u32 ad, u16 v){
	asmv("outw %%ax, %%dx" :: "d" (ad), "a" (v));
}
/* output word */
void Io::outl(u32 ad, u32 v){
	asmv("outl %%eax, %%dx" : : "d" (ad), "a" (v));
}
/* input byte */
u8 Io::inb(u32 ad){
	u8 _v;       \
	asmv("inb %%dx, %%al" : "=a" (_v) : "d" (ad)); \
	return _v;
}
/* input word */
u16	Io::inw(u32 ad){
	u16 _v;			\
	asmv("inw %%dx, %%ax" : "=a" (_v) : "d" (ad));	\
	return _v;
}
/* input word */
u32	Io::inl(u32 ad){
	u32 _v;			\
	asmv("inl %%dx, %%eax" : "=a" (_v) : "d" (ad));	\
	return _v;
}
```