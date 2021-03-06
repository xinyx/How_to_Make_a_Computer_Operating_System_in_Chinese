## 第八章: 内存管理:物理内存和虚存

在与GDT相关的一章中, 我们看到使用段机制, 物理内存地址要用一个段选择子和一个偏移计算. 

在本章, 我们将实现分页. 分页将线性地址从段映射到物理地址. 

#### 我们为什么要分页?

分页机制允许我们的内核: 

* 像内存一样使用硬盘驱动器, 并且不会受限于机器的ram容量.
* 为每个进程分配独有的内存空间. 
* 动态控制内存空间是否有效. 

在分页系统中, 每个进程可以执行在它的的4Gb内存中, 绝不会影响其他进程或内核的内存. 它简化了多任务调度.


![Processes memories](./processes.png)

#### 它是怎么工作的?

从线性地址到物理地址的映射需经过多步:

1. 处理器通过`CR3`寄存器知道页目录表的起始地址.
2. 线性地址的前十位表示一个偏移(0~1023), 指向页目录表中的一项. 这一项包含一个页表的物理地址. 
3. 线性地址接下开的10位也表示一个偏移, 指向页表中的一项, 这一项指向一个4KB页帧.
4. 线性地址的最后12位表示偏移(0~4095), 表示在4KB页帧中的位置.

![Address translation](./paging_memory.png)

#### 格式化你的页表和页目录表

这两个类型的表项(表和目录)看起来是一样的. 我们的操作系统只会使用标灰的区域. 

![Page directory entry](./page_directory_entry.png)

![Page table entry](./page_table_entry.png)

* `P`: 指示这个页或者表是否在物理内存中
* `R/W`: 指示这个页或者表能否写(值等于1)
* `U/S`: 等于1时允许访问非首选任务
* `A`: 指示这个页或页表有没有被访问
* `D`: (只对页表有意义)指示页有没有被写
* `PS` (只对页目录表有意义)指示页大小
    * 0 = 4ko
    * 1 = 4mo

**注解:** 页目录和页表中的物理地址用20位表示, 这是因为这些地址是4KB对齐的, 所以后12位等于0.

* 页目录和页表占用 1024*4 = 4096 bytes = 4k
* 一个页表能寻址 1024 * 4k = 4 Mo
* 一个页目录能寻址 1024 * (1024 * 4k) = 4 Go

#### 怎样开启分页?

为了开启分页, 只需设置 `CR0` 的第31位为1即可.

```asm
asm("  mov %%cr0, %%eax; \
       or %1, %%eax;     \
       mov %%eax, %%cr0" \
       :: "i"(0x80000000));
```

但是在之前, 我们需要至少一个也来初始化我们的页目录. 



