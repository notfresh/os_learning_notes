# 虚拟内存
## TLB异常处理
### Exercise1 : 源代码阅读
code/userprog/addrspace.cc   
与地址空间管理相关的代码。在class Thread 中进程有一个私有变量AddrSpace * space, 可知每个进程有一个独立的地址空间。  

* AddrSpace::AddrSpace() 新建一个页表，并完成对页表初始化。此时virt page i = phys page i。
* AddrSpace::Load() 将用户程序加载到内存。  
    1. 打开文件
    2. 大小端转换(NOFFMAGIC的作用？)
    3. 计算文件需要占用多少页，检查是否超过最大页数
    4. 按照**noff Header**中的信息，将文件中的段读到主存对应的虚拟地址
* AddrSpace::InitRegisters()  初始化用户态下将要使用的寄存器
* AddrSpace::RestoreState() 恢复页表
* AddrSpace::Execute() 调用machine::Run()执行用户代码
* AddrSpace::Translate() 进行地址转换。
    1. 根据虚拟地址计算虚拟页号和偏移
    2. 根据虚拟页号从页表中取出对应的项
    3. 根据PTE得到对应的物理页框号
    4. 物理地址 = 物理页框号 * PageSize + offset

code/machine/machine.h(cc)  
定义执行用户代码的虚拟机。  

虚拟机包括：  
* 主存
* TLB
* 页表
* 寄存器

code/machine/translate.h(cc)  
TranslationEntry是页表或者TLB中的一项，其结构为：  
| 逻辑页号 | 物理页号 | valid | readonly | use | dirty |  

Machine::Translate()  
![地址转换流程.JPG](https://i.loli.net/2019/10/29/JmBW4rn8fGScyL6.jpg)  
问题
1. 为什么页表和TLB不能共存？
2. Machine::Translate和AddrSpace::Translate有什么不同？
***
code/userprog/exception.h(cc)  
中断|异常处理函数,根据中断号进行相应的处理  
***
### Exercise2 : TLB MISS异常处理 | Exercise3 : 置换算法
下面给出改进之后的地址转换流程图：  
![KXJCDJ.png](https://s2.ax1x.com/2019/11/03/KXJCDJ.png)  
code/machine/translate.h  
为了方便对TLB进行管理，定义一个class TLBEntry和一个class TLB。  
TLBEntry定义TLB中的一项，包括：  
* Tag
* PPN
* valid
* lru  

TLB结构为4*4，包括一个成员变量TLBEntry * tlbPtr[4]，它是一个指针数组，每个指针指向一行(4个)TLBEntry数组。  
类TLB有两个方法：translate()和update。下面给出两个方法实现的具体思路。
* int translate(int virtAddr)：根据虚拟地址查TLB，进行地址翻译。  
    * 根据虚拟地址计算vpn
    * 根据vpn计算TLBI(index, 用于定位行)和TLBT(Tag,用于在一行之内进行比较)
    * 遍历TLB的第TLBI行，如果该项的valid为TRUE且Tag与TLBT相同，则hit。取出PPN，计算出物理地址，返回;否则miss 返回-1.  

* void TLB::update(int virtAddr, int pageFrame)：更新TLB.  
    * 根据虚拟地址计算vpn、TLBI、TLBT
    * 找第TLBI行是否有空闲项，如果没有根据lru算法找出一项
    * 替换该项(令PPN = pageFrame，填写Tag等)  
***
## 分页式内存管理  
当前Nachos内存管理的思路是：main.cc最后通过Addrspace::Load()函数将用户态程序一次性加载到内存中(这里内存是分段的)。而且虚拟地址和物理地址是相等的。要想支持分页式内存管理、支持多进程，主要实现以下几点：  
* 对物理内存进行分页管理
* 利用位图记录物理内存分配情况
* 维护页表
* 缺页中断处理
### Exercise4 : 内存全局管理数据结构  
思路：基于Nachos中已经提供的bitmap实现内存全局管理。bitmap中的每一位代表一个物理页框，为1代表已分配，为0代表未分配。    
为了体现虚拟存储的优越性，在code/machine/memory.h中规定物理页有128个，虚拟页有1024个。所以bitmap有128位。  
code/machine/machine.h ：  
* 为class machine增加成员变量Bitmap * memoryMap
* 对Machine构造函数和析构函数进行修改。在构造函数中new Bitmap，在析构函数中进行delte 
* 在进行物理页的分配时使用memoryMap->FindAndSet()函数找到一个空闲页
* 在进行物理页的回收时利用memoryMap->Clear()函数对相应的位进行擦除。
***
### Exercise5 : 多线程支持
在实现了虚存和分页的基础上，支持多线程就变得非常容易。因为物理内存不再是被一个进程独占，而是通过页表将进程的虚拟地址中的页与物理地址中的页进行了映射。所以此时可以将多个文件同时加载到内存中，从而支持了多线程。
***
### Exercise6 : 缺页中断处理
code/userprog/exception.cc ： 在ExceptionHandler()中通过对异常类型判断执行相应的处理操作。如果当前为PageFaultException，则执行PageFaultHandler()。  
PageFaultHandler()主要是通过调用AddrSpace::LoadOnePage()**将发生缺页的虚拟地址所在的一页**从磁盘上加载到内存中。  
AddrSpace::LoadOnePage(int VAddr)处理流程：  
1. 打开当前地址空间所对应的用户程序文件
2. 找到一个物理页框
3. 填写页表
4. 通过OpenFile::ReadAt()函数将磁盘上的一页读到内存中。  

这里有一个问题：给定一个虚拟地址，我们可以计算出该虚拟地址所在的页号，但是这个页号是虚拟地址空间中的逻辑页号，我们如果根据虚拟页号找到该页所对应的磁盘上用户程序的那一页？  
这里我假设用户程序(除Header)直接从虚拟地址为0开始加载到虚拟内存中. 
```c++
executable->ReadAt(&(kernel->machine->mainMemory[physPage * PageSize]), PageSize, virtPage * PageSize + sizeof(NoffHeader));
```
***
## Lazy-loading
有了分页机制、虚拟内存、缺页中断处理，我们就可以实现lazy-loading。  
更改code/threads/main.cc，如果userProgName!=NULL，则建立地址空间，直接执行space->Excute()。此时用户程序没有加载到内存中，在执行第一条指令的时候，查TLB,miss，查页表,miss。此时会pageFault，然后通过中断处理函数将磁盘上的一页读进来。这就是lazy-loading。
***
## Challenges
尚未实现











