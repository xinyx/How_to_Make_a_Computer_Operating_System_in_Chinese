## 第七章: IDT和中断

中断是硬件和软件向处理器触发的信号, 指示一个需要立即引起注意的事件.

有三种类型的中断:

- **硬件中断:** 由外设(键盘, 鼠标, 硬盘, ...)发送给处理器. 硬件中断被引入是为了减少处理器轮询和等待外部事件的宝贵时间.
- **软件中断:** 自动地由软件初始化, 用来管理系统调用.
- **异常:**  用来处理程序在执行过程中出现的错误或事件, 这些错误或事件很异常, 不能由程序自身处理(如除零, 页错误, ...). 

#### 一个键盘的例子:

当用户在键盘上按键时, 键盘控制器将向中断控制器发送一个中断. 如果这个中断没有被屏蔽, 中断控制器将向处理器发送一个中断, 处理器将执行一个例程来管理中断(按键或释放键), 例如, 这个例程可以从键盘控制器获得被按的键并将其打印到屏幕上. 一旦完成字符处理例程, 被中断的工作就可以继续了. 

#### 什么是PIC?

[PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller) (可编程中断处理器)是一个设备, 用于结合多个中断源到一个或多个CPU总线上, 同时允许为中断输出分配优先级. 当设备要发送多个中断输出时, 需要按照相关优先级依次进行. 

最注明的PIC是8259A, 每个8259A可以处理8个设备, 但是多数计算机都有两个控制器:一个主控制器, 一个从控制器. 这能使计算机管理来自14个设备的中断.

本章我们需要为这个控制器编程来初始化和屏蔽中断.

#### 什么是 IDT?

> 中断描述符表(IDT)是x86架构使用的一种数据结构, 用来实现一个中断向量表. 处理器使用IDT来决定对中断和异常的正确回应.

我们的kernel将使用IDT定义中断发生时需要执行的多个函数.

与GDT一样, IDT被汇编指令LIDTL加载. 它期望的IDT描述结构的地址如下:

```cpp
struct idtr {
	u16 limite;
	u32 base;
} __attribute__ ((packed));
```

The IDT table is composed of IDT segments with the following structure:

```cpp
struct idtdesc {
	u16 offset0_15;
	u16 select;
	u16 type;
	u16 offset16_31;
} __attribute__ ((packed));
```

**注意:** “__attribute__((packed))“指令会通知gcc, 应该尽可能使用小端方式使用这个结构。如果没有这个指令, gcc将加进一些字节来优化内存对齐和期间执行的访问. 

现在我们需要定义我们的IDT表并且用LIDTL加载它. IDT可以存储在内存的任何位置, 只需使用IDTR寄存器将它的地址发送给进程即可.

这是一个常见中断的表格(可屏蔽硬件中断被成为IRQ):


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

#### 怎样初始化中断?

这是一个定义IDT段的简单例子.

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

现在我们可以初始化这个中断. 

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

初始化IDT后, 我们需要配置PIC来激活中断. 以下函数将配置两个PIC, 这是通过使用处理器的输出端口```io.outb```来写入其内部寄存器实现的. 我们用这些端口配置PIC: 

* Master PIC: 0x20 and 0x21
* Slave PIC: 0xA0 and 0xA1

对于一个PIC来说, 有两个类型的注册过程: 

* ICW (初始化命令字): 重新初始化控制器
* OCW (操作控制字): 一旦初始化便配置控制器(用来屏蔽/打开中断)

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

注册过程必须按顺序执行.

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

主控制器:
```
|x|x|x|x|x|x|x|x|
 | | | | | | | |
 +------------------ slave controller connected to the port yes (1), or no (0)
```

从控制器:
```
|0|0|0|0|0|x|x|x|  pour l'esclave
           | | |
           +-------- Slave ID which is equal to the master port
```

**ICW4 (port 0x21 / port 0xA1)**

用来定义控制器应该工作在什么模式.

```
|0|0|0|x|x|x|x|1|
       | | | +------ mode "automatic end of interrupt" AEOI (1)
       | | +-------- mode buffered slave (0) or master (1)
       | +---------- mode buffered (1)
       +------------ mode "fully nested" (1)
```

#### 为什么idt段抵消了我们的ASM函数?

你应该注意到了, 当我初始化我们的IDT段时, 我在汇编语言中使用偏移来分段代码. 不同的函数定义在[x86int.asm](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/x86int.asm), 用了以下实施方案:

```asm
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

这些宏将用来定义中断段, 防止不同的注册过程被打断, 这在多任务中很有用.
