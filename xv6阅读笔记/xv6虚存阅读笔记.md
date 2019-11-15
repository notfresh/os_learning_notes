# xv6虚存阅读笔记
## 开机启动过程简述
### BIOS
* 将硬盘的第一个扇区（引导扇区，存放的是bootasm.S）加载到内存的0x7c00处。
***
### bootasm.S
* 开启A20 : 设置8042芯片的输出端口(0x64)
* 从实模式转为保护模式 : 置CR0的PE位，并使用一个ljmp指令设置cs和eip
* 初始化段寄存器和esp

asm.h中宏SEG_ASM(type, base, lim)的作用是定义一个段。
```c
#define SEG_ASM(type,base,lim)                                  \
        .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);      \
        .byte (((base) >> 16) & 0xff), (0x90 | (type)),         \
                (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
```
在bootasm.S中定义了1个GDT，其中包括3个LDT
```x86asm
gdt:
  SEG_NULLASM                             # null seg
  SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)   # code seg
  SEG_ASM(STA_W, 0x0, 0xffffffff)         # data seg

gdtdesc:
  .word   (gdtdesc - gdt - 1)             # sizeof(gdt) - 1
  .long   gdt                             # address gdt
```
gdtdesc是GDT描述符，其作用是找到GDT。lgdt命令将GDT描述符加载到寄存器GDTR中。

**保护模式下地址转换：逻辑地址->虚拟地址(线性地址)->物理地址**  
逻辑地址转换为虚拟地址示意图：

![Md4arR.jpg](https://s2.ax1x.com/2019/11/15/Md4arR.jpg)

***
### bootmain.c
* 将内核从磁盘加载到内存

具体过程：编译好的内核以ELF文件的形式存放在磁盘上。首先从磁盘上读ELF文件的ELF header，然后根据ELF header找到program header，最后根据各个program header将各个段加载到内存中。 
***
### entry.S
* 开启页扩展，方法：设置CR4中的page size extension位，扩展后每页大小为4MB
* 设置页目录(page directory)，方法：将页目录地址写入CR3。
* 开启分页机制，方法：设置CR0中的PG位

此时使用的页目录是entrypgdir,其定义在main.c中
```c
__attribute__((__aligned__(PGSIZE)))
pde_t entrypgdir[NPDENTRIES] = {
  // Map VA's [0, 4MB) to PA's [0, 4MB)
  [0] = (0) | PTE_P | PTE_W | PTE_PS,
  // Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
  [KERNBASE>>PDXSHIFT] = (0) | PTE_P | PTE_W | PTE_PS,
};
```
entrypgdir这个页目录中有两个页表项。
***
### main.c
1. kinit1(end, P2V(4*1024*1024)) : 为虚拟地址[end, P2V(4*1024*1024)]分配物理内存
2. kvmalloc(void)
    * kpgdir = setupkvm()
        * 调用kalloc()为页目录申请一页物理内存
        * 调用mappages为kmap中各个段创建PTE并为对应页表分配内存空间(walkpgdir)
    * switchkvm() : 加载kpgdir
3. seginit(void)
4. kinit2(P2V(4*1024*1024), P2V(PHYSTOP)) : 为虚拟地址[P2V(4*1024*1024), P2V(PHYSTOP)]分配物理内存

重点函数具体分析：

freerange(void* vstart, void* vend) ：为虚拟地址[vstart, vend]分配物理内存  
* 计算所需物理页数
* 循环调用kfree分配一页空间  

kfree(char* v) ： 为v所指向的一页分配物理内存(注意：这里v指向一页的页首地址)  
* memset(v, 1, PGSIZE) 将这一页全部置为1(dangling references)，好处：cause code that uses memory after freeing it to break faster。
* 将当前页放到kmem.freelist的队首  

pte_t* walkpgdir(pde_t* pgdir, const void* va, int alloc) : 返回va在页的PTE，如果alloc不为0，需要的情况下创建PTE，申请一个物理页作为页表
1. 使用PDX宏获得va所对应的PTE在页目录中的下标
2. 如果PTE的存在位为1，通过PTE_ADDR宏获得PTE所在的地址(物理)，通过p2v将其转化为虚拟地址
3. 如果存在位为0且alloc位非0，则申请一页用作页表，并填写PTE
4. 返回PTE在页目录中的地址

问题：struct run* r = (struct run*)v,这是什么骚操作？  
一图胜千言

![Mwk7M4.png](https://s2.ax1x.com/2019/11/16/Mwk7M4.png)

后续将空闲页分配出去之后它就不属于空闲页了，所以struct run也没用了，不用担心对struct run这片内存的写会出现问题。

## 虚拟内存布局
![MweP9P.jpg](https://s2.ax1x.com/2019/11/16/MweP9P.jpg)

几个问题：
* 虚拟地址[0, KERNBASE]映射到哪里？
* 如何理解这张图？
***
## 动态内存管理
上面解释过。
***
## 页式管理
* 发生中断时使用哪个页表？
* 一个页多大？
* 页目录有多少项？页表有多少项？最大支持多大内存？
* 虚拟地址到物理地址的转换图？
![MweOCq.jpg](https://s2.ax1x.com/2019/11/16/MweOCq.jpg)

* 如何实现虚拟地址到物理地址的映射？
