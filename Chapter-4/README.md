## 第四章: 操作系统的主干和C++运行时

#### C++ kernel运行时

内核可以用c++编程, 它非常类似于用C做一个内核, 但是有几个陷阱你必须考虑(运行时支持, 构造函数…). 

编译器假定所有必要的c\+\+运行时支持都是默认可用的, 但因为我们不连接libsupc\+\+到c\+\+的内核, 所以我们需要添加一些基本功能, 这可以在[cxx.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/cxx.cc)中找到. 

**注意:** 在初始化虚存和分页之前, 不能使用操作子 `new` 和 `delete`. 

#### 基本的C/C\+\+函数

内核代码不能使用标准库函数,所以我们需要添加一些基本函数用于管理内存和字符串:

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

这些函数定义在[string.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/string.cc), [memory.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/memory.cc), [itoa.cc](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/itoa.cc)中. 

#### C数据类型

在接下来的步骤中,我们将在我们的代码中使用不同类型, 我们将使用的大部分的类型是无符号类型(所有的位都用于存储整数, 在有符号类型中有一位用于符号标记):

During the next step, we are going to use different types in our code, most of the types we are going to use unsigned types (all the bits are used to stored the integer, in signed types one bit is used to signal the sign):

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

编译内核不同于编译linux可执行文件, 我们不能使用标准库, 并且应该独立于系统. 

我们的[Makefile](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/Makefile)将定义编译和链接我们的操作系统的过程. 


对于x86体系结构, gcc/g++/ld会使用下面这些参数: 

```makefile
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
