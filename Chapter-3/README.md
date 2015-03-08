## 第三章: GRUB的初次启动

#### 到底怎么启动的?

当打开一台基于x86的计算机时, 它便开始一条复杂的路径到达某个阶段, 在这里控制权转移到内核的“main”程序(“kmain()”). 对于这门课程,我们只会考虑BIOS启动方法,而不是它的继任者(UEFI)。

BIOS的启动顺序是: RAM探测 -> 硬件检测/初始化 -> 启动顺序. 

对我们来说, 最重要的一步是启动顺序, 其中BIOS已经做好了初始化工作并且尝试将控制移交至之后的bootloader程序. 

在“启动顺序”阶段, BIOS将试图确定“启动设备”(如软盘、硬盘、光盘、USB闪存设备或网络). 我们的操作系统将首先从硬盘启动(但以后也可能从CD或USB闪存设备引导). 如果一个设备在511和512偏移处分别包含有效的签名字节`0x55`和`0xAA`(称作主引导记录(MBR), 这个签名的二进制表示是0b1010101001010101. 用这种交替的位模式是为了防止某些错误(驱动器或控制器). 如果这个值被篡改或为0x00, 则设备被认为是不可引导的). 

BIOS身体上搜索一个引导设备通过加载第一个512字节的引导区每个设备到物理内存,从地址0 x7c00”(1简约低于32简约标志)。当检测到字节的有效签名,BIOS将控制转移到0 x7c00的内存地址(通过跳转指令)来执行引导区代码。

BIOS通过加载每个设备的引导扇区的前512字进入物理内存的0x7C00(32KB掩码下是1KB)处, 来物理搜索启动设备. 一旦检测到有效的签名字节, BIOS将控制移交至物理内存的`0x7C00`处(通过一个跳转指令)来执行引导扇区代码. 

通过这个过程, CPU将运行在16位实模式(x86的默认状态, 以便实现向后兼容). 为了在我们的内核中执行32位指令, 我们需要bootloader来切换CPU至保护模式. 

#### 什么是GRUB?
> GNU GRUB(GNU大型统一引导装载程序的简称)是GNU项目的引导加载程序包. GRUB是自由软件基金会的多重引导规范的参考实现, 它使得用户可以从安装在计算机的多个操作系统中选择某一个或者从特定操作系统的分区中选择某种存在的内核配置.

为简单起见, GRUB是机器(boot-loader)第一次启动的程序, 并且可以简化硬盘中内核的加载过程.

#### 我们为什么使用GRUB?

* GRUB易于使用
* 简化不使用16位代码加载32位内核的过程
* Linux, Windows和其他操作系统的多重启动
* 简化加载内存中的外部模块

#### 怎么使用GRUB?

GRUB使用多重引导规范, 可执行二进制程序应该是32位, 并且在它的前8192字节中必须包含一个特殊的头(多重引导头). 我们的内核必须是一个ELF可执行文件(“可执行的和可链接格式”, 大多数UNIX系统中的可执行文件的一个通用标准文件格式)。

我们的kernel的第一个启动步骤使用汇编写的: [start.asm](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/start.asm) 并且我们用一个链接文件来定义我们的可执行结构体: [linker.ld](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/linker.ld).

这步启动过程也初始化了部分C++运行时, 这将在下一章描述.

多重引导头结构: 

```cpp
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

你可以用命令 ```mbchk kernel.elf``` 来用多重引导规范验证你的kernel.elf文件. 也可以用命令 ```nm -n kernel.elf``` 来验证ELF二进制文件中不同元素的偏移.

#### 为我们的kernel和grub创建磁盘镜像

脚本[diskimage.sh](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/sdk/diskimage.sh)将生成一个可被QEMU使用的硬盘镜像. 

第一步是用qemu-img创建一个硬盘镜像(c.img): 

```
qemu-img create c.img 2M
```

现在我们需要用fdisk为磁盘分区: 

```bash
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

我们现在需要使用losetup把创建的分区附到循环设备(允许一个文件像块设备那样被访问). 分区的偏移量作为参数, 这样计算: **offset=start_sector * bytes_by_sector **. 

Using ```fdisk -l -u c.img```, you get: 63 * 512 = 32256.

```bash
losetup -o 32256 /dev/loop1 ./c.img
```

我们在这个新设备上创建EXT2文件系统:

```bash
mke2fs /dev/loop1
```

复制文件到一个挂载磁盘:

```bash
mount  /dev/loop1 /mnt/
cp -R bootdisk/* /mnt/
umount /mnt/
```
在这个磁盘上安装GRUB: 

```bash
grub --device-map=/dev/null << EOF
device (hd0) ./c.img
geometry (hd0) 4 16 63
root (hd0,0)
setup (hd0)
quit
EOF
```

最后我们分离块设备: 

```bash
losetup -d /dev/loop1
```

#### 另请参阅

* [维基百科GNU GRUB](http://en.wikipedia.org/wiki/GNU_GRUB)
* [多重引导规范](https://www.gnu.org/software/grub/manual/multiboot/multiboot.html)
