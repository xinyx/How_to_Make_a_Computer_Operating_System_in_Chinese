## 第六章: GDT

多亏了GRUB, 你的内核不再是在实模式, 而是早已在[保护模式](http://en.wikipedia.org/wiki/Protected_mode), 这个模式使我们能够使用微处理器的所有的可能功能, 如虚拟内存管理、分页和安全的多任务. 

#### 什么是GDT?

[GDT](http://en.wikipedia.org/wiki/Global_Descriptor_Table) ("全局描述符表")是一种用于定义不同的内存区域--基地址、大小和访问权限, 例如执行,写--的数据结构。这些内存区域被称为“段”. 

我们将使用GDT来定义不同的内存段: 

* *"code"*:内核代码, 用于存储可执行二进制代码
* *"data"*: 内核数据
* *"stack"*: 内核栈, 用于存储内核执行过程中的调用栈
* *"ucode"*: 用户代码, 用于存储用户程序的可执行二进制代码
* *"udata"*: 用户程序数据
* *"ustack"*: 用户栈, 用于存储用户态执行过程中的调用栈

#### 怎样加载GDT?

GRUB初始化一个GDT, 但这GDT并不对应于我们的内核. GDT是被LGDT这条汇编指令加载的. 它期望GDT描述结构的位置为:

![GDTR](./gdtr.png)

C结构是:

```cpp
struct gdtr {
	u16 limite;
	u32 base;
} __attribute__ ((packed));
```

**注意:** “__attribute__((packed))“指令会通知gcc, 应该尽可能使用小端方式使用这个结构。如果没有这个指令, gcc将加进一些字节来优化内存对齐和期间执行的访问. 

现在我们需要定义GDT表, 然后使用LGDT加载它. GDT表可以存储在内存中的任何位置, 只需使用GDTR寄存器将它的地址发送给进程即可.

GDT表由具有如下结构的段组成: 

![GDTR](./gdtentry.png)

C结构如下: 

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

#### 怎样定义我们的GDT表?

现在我们需要在内存中定义我们的GDT, 并最终用GDTR寄存器将其加载. 

我们将GDT存在如下地址:

```cpp
#define GDTBASE	0x00000800
```

 [x86.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/x86.cc)中的函数**init_gdt_desc**初始化hi个gdt段描述符. 

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

函数**init_gdt** 初始化GDT, 下面函数的一些部分将在后面解释, 并用于多任务.

```cpp
void init_gdt(void)
{
	default_tss.debug_flag = 0x00;
	default_tss.io_map = 0x00;
	default_tss.esp0 = 0x1FFF0;
	default_tss.ss0 = 0x18;

	/* initialize gdt segments */
	init_gdt_desc(0x0, 0x0, 0x0, 0x0, &kgdt[0]);
	init_gdt_desc(0x0, 0xFFFFF, 0x9B, 0x0D, &kgdt[1]);	/* code */
	init_gdt_desc(0x0, 0xFFFFF, 0x93, 0x0D, &kgdt[2]);	/* data */
	init_gdt_desc(0x0, 0x0, 0x97, 0x0D, &kgdt[3]);		/* stack */

	init_gdt_desc(0x0, 0xFFFFF, 0xFF, 0x0D, &kgdt[4]);	/* ucode */
	init_gdt_desc(0x0, 0xFFFFF, 0xF3, 0x0D, &kgdt[5]);	/* udata */
	init_gdt_desc(0x0, 0x0, 0xF7, 0x0D, &kgdt[6]);		/* ustack */

	init_gdt_desc((u32) & default_tss, 0x67, 0xE9, 0x00, &kgdt[7]);	/* descripteur de tss */

	/* initialize the gdtr structure */
	kgdtr.limite = GDTSIZE * 8;
	kgdtr.base = GDTBASE;

	/* copy the gdtr to its memory area */
	memcpy((char *) kgdtr.base, (char *) kgdt, kgdtr.limite);

	/* load the gdtr registry */
	asm("lgdtl (kgdtr)");

	/* initiliaz the segments */
	asm("   movw $0x10, %ax	\n \
            movw %ax, %ds	\n \
            movw %ax, %es	\n \
            movw %ax, %fs	\n \
            movw %ax, %gs	\n \
            ljmp $0x08, $next	\n \
            next:		\n");
}
```
