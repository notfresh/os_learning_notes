# 文件系统
<p align="right">撰写人：高凡斐</p>

## UNIX文件系统的主要组成部分
### 整体布局
unix文件系统的整体布局
![QJceKI.png](https://s2.ax1x.com/2019/12/06/QJceKI.png)  
![QJcWdK.png](https://s2.ax1x.com/2019/12/06/QJcWdK.png)
xv6文件系统的整体布局
![QtPG28.png](https://s2.ax1x.com/2019/12/07/QtPG28.png)
### 超级块
**文件系统的元数据**，定义了文件系统的类型、大小、状态等。  
```c
struct superblock {
  uint size;         // Size of file system image (blocks)
  uint nblocks;      // Number of data blocks
  uint ninodes;      // Number of inodes.
  uint nlog;         // Number of log blocks
};
```
### i节点
存放**文件的元数据**，如文件大小、文件类型等。  
```c
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEV only)
  short minor;          // Minor device number (T_DEV only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```
在linux中可以使用命令stat查看一个文件的inode信息  
![QJyNSs.jpg](https://s2.ax1x.com/2019/12/06/QJyNSs.jpg)  
i节点也会消耗磁盘的空间，所以硬盘格式化的时候会分一个inode区专门用于存放inode。inode节点的个数在格式化时就确定了，一般是每1KB或者每2KB设置一个inode。由于每个文件都必须有一个inode，因此**可能发生inode已经用光，但是磁盘还未存满的情况**，此时无法在磁盘上创建新文件。
### 数据块
存放文件的数据。

### 目录块
存放目录文件的数据块。目录是由一个个目录项构成，每个目录项包含一个文件名和它对应inode的编号。  
```c
struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```
### 间接块
为了解决大文件存储问题，建立索引表，存放索引表的那些数据块称为间接块。
[![QJcXo8.md.png](https://s2.ax1x.com/2019/12/06/QJcXo8.md.png)](https://imgse.com/i/QJcXo8)
***
## 概述
xv6文件系统由7层组成：
* Disk：在一个IDE硬件设备上读写块。
* Buffer cache: 缓存硬盘上的块、同步对磁盘块的各种操作，保证各个内核进程互斥地修改磁盘块。
* Logging: 允许上层将对多个块的操作放在一个事务(transaction)中，保证在crash的时候对块的更新是原子的。
* Inode：支持文件的概念，每个文件由一个inode和保存该文件的块构成。
* Directory：将每个目录作为一种特殊的inode，它的内容由一系列目录项构成，目录项包含文件名和i-number。
* pathname：支持文件路径和对文件路径的递归解析
* File descriptor：对unix的各种资源(如管道、设备、文件等)加以抽象，简化应用程序的编程。  
![QJg2lj.png](https://s2.ax1x.com/2019/12/06/QJg2lj.png)
***
## Disk
设备驱动层。主要是驱动程序，使用汇编操作具体的外设进行读写操作。
***
## Buffer cache
Buffer cache位于磁盘和上层文件系统之间，主要作用是缓存那些经常使用的块。  
xv6的buffer cache使用一个数据结构bcache来表示.  
```c
struct {
  struct spinlock lock; //互斥访问bcache
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // head.next is most recently used.
  struct buf head;
} bcache;
```
链表中的元素为结构体buf，**每个buf同时只能被一个进程占有**，占有与否使用flags中的B_BUSY位来表示。当一个进程想要获取某个buf时会检查它的B_BUSY位，如果B_BUSY位为1，则睡眠；否则成功，设置B_BUSY位。当一个进程使用完buf之后需要调用brelease位清空B_BUSY位。  
```c
struct buf {
  int flags;    //buffer的状态
  uint dev;
  uint sector;
  struct buf *prev; // LRU cache list
  struct buf *next;
  struct buf *qnext; // disk queue
  uchar data[512];
};
```

链表中buf的个数是固定，为NBUF。  
bcache对外提供的接口包括:
* binit : 初始化bcache
* bget(uint dev, uint sector) : 获得指定设备上的指定扇区的buf
* bread(uint dev, uint sector) : 返回一个包含指定扇区内容的buf
* bwrite(struct buf *b) : 将一个buf写回磁盘
* brelse(struct buf *b) : 释放一个buf

### binit
初始化bcache，主要是使用**头插法**建立双向链表。
![QJq6Qx.png](https://s2.ax1x.com/2019/12/06/QJq6Qx.png)
### bget
函数功能：获得一个buf来存放指定设备指定扇区的内容。如果该扇区已经被读入bcache中的某个buf，则睡眠等待该buf直到该buf空闲；否则找一个空闲的buf(B_BUSY和B_DIRTY都为0),并初始化该buf。**没有将磁盘上的内容读到buf中**    
函数参数：设备号、扇区号  
返回值：static struct buf*  
函数实现：主要是对链表进行两次遍历
* 对bcache加锁
* 第一次遍历，检测sector是否已经被cache
    * 当前buf缓存了指定的sector
        * buf没有被占有，则占有该buf、释放bcache锁并成功返回
        * buf被占有，则sleep
* 第二次遍历，寻找一个空闲buf
    * 找到一个没有别占用并且不含脏数据的buf，设置该buf、释放bcache锁并成功返回
* 没有剩余buf,报panic
### bread
函数功能：将指定设备指定扇区的内容从磁盘读到buffer cache中。  
函数参数：设备号、扇区号
返回值：static struct buf*  
函数实现：首先调用bget函数获得一个buf，然后判断sector的数据是否已经读到该buf中，如果已经读入buf中则直接返回；否则调用irew读磁盘然后返回。
### bwrite
函数功能：将指定的buf写回到磁盘  
函数参数：static struct buf* b  
返回值：无  
函数实现：首先判断buf是否被占用，如果没有则报错。然后设置B_DIRTY位，并调用iderw函数进行写回(在iderw中会清空B_DIRTY)
### brelease
函数功能：释放一个B_BUSY的buffer  
函数参数：static struct buf* b  
返回值：无  
函数实现：首先确保b是B_BUSY的。然后使用头插法将该buf插入到双向循环链表里。最后清空B_BUSY位，并调用wakeup函数唤醒等待该buf的进程。
***
## Logging
日志层主要的作用是错误恢复，解决掉电等导致的不一致问题。主要思想是将用户对磁盘的多次操作包装在一个事务(transaction)中，先将要写入目的块的数据写到一个日志块中，并在log header的sector数组中记录目的块，最后一并将日志块的数据写到目的块中。
![QtiTkn.png](https://s2.ax1x.com/2019/12/07/QtiTkn.png)
主要涉及两个数据结构：  
logheader
```c
struct logheader {
  int n;   
  int sector[LOGSIZE];
};
```
log:日志管理的数据结构
```c
struct log {
  struct spinlock lock;
  int start;    //log开始扇区(存放logheader)
  int size;     //log所占总扇区
  int busy; // a transaction is active
  int dev;
  struct logheader lh;
};
```
### initlog
函数功能：日志模块初始化。  
函数实现：主要是读superblock，并根据superblock初始化log结构体。然后调用recover_from_log函数。
### install_trans
函数功能：将日志块中暂存的数据copy到它们应该在的磁盘块。  
函数实现：使用bread函数将日志块的数据从磁盘读入lbuf，将目的磁盘块的数据从磁盘读入dbuf，然后在内存中将lbuf的内容拷贝给dbuf，最后使用bwrite将dbuf写入磁盘。
### read_head
函数功能：将log header从磁盘读入内存。
### write_head
函数功能：将log header从内存写入磁盘。
### recover_from_log
函数功能：根据日志进行恢复。  
函数实现：读log header，并执行install_trans函数。  
### begin_trans
函数功能：保证各个进程互斥使用log机制  
函数实现：主要是进行加锁操作。首先对log结构体加锁，然后判断log是否正在被使用，如果被使用则调用sleep等待；否则设置busy对log加锁。最后释放log结构体的锁。
### commit_trans
函数功能：提交一次transaction
函数实现：首先判断log中未提交的日志块的数目，如果大于0，则**首先将log header写回磁盘**，然后执行install_trans函数将各个日志块写到目的块，然后设置log header的n为0，并再次将log header写回磁盘(此时相当于将各个日志块变为free)，最后释放log。
### log_write
函数功能：将用户想要写到磁盘中的数据先写到一个日志块中
函数参数：struct buf *b
函数实现：首先确保当前还有free的日志块，然后查看各个待写入日志块，如果有日志块与b的sector相等，则将b的内容写入该日志块中;否则将b的内容写入到一块新的日志块，并设置log header.  

log机制使用套路：
* begin_trans()
* 一系列writei()
* commit_trans()将日志块写到目的块
***
## Inode
一个inode节点描述了一个无名的文件(文件名在目录中)。inode节点内存放了文件的元数据(metadata)，比如：文件的长度、文件类型、链接数、数据块地址等。  
xv6中inode对应的数据结构有两种，一个是存放在磁盘上的inode，一个是在内存中的inode。  
```c
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEV only)
  short minor;          // Minor device number (T_DEV only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses +1代表间接块地址
};
```
```c
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  int flags;          // I_BUSY, I_VALID

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```
可以看到，dinode包含了文件的一些静态属性，而inode还包含了一些动态属性，比如链接数、状态等。  
同时，xv6使用一个结构体icache保存活跃的inode。
```c
struct {
  struct spinlock lock;
  struct inode inode[NINODE];
} icache;
```
在inode结构体中最重要的是数据块地址，xv6中使用一个addr数组来存放数据块对应的地址。每个inode包含NDIRECT个直接盘块和一个间接盘块。所以xv6中文件最大为NDIRECT + NINDIRECT个块，其中NINDIRECT = (BSIZE / sizeof(uint))。  
[![QtEe74.md.png](https://s2.ax1x.com/2019/12/07/QtEe74.md.png)](https://imgse.com/i/QtEe74)  

相关函数包括
* iinit : icache初始化
* ialloc : 在指定设备上申请一个空闲inode，并设置其类型
* iupdate : 将内存中的inode写到磁盘上(使用log_write)
* iget : 获得指定设备号指定inum的inode
* idup : 将给定inode的引用数+1
* ilock : 对给定inode加锁，如果其不在内存则读入内存
* iunlock : 对给定inode解锁
* iput : 将给定inode的引用书-1，并在必要时回收inode
* iunlockput : 解锁并put
* bmap : 返回inode中    第n个数据块的地址，如果没有的话就创建一个
* itrunc : 释放inode占用的数据块
* stati : copy inode中的信息
* readi : 根据inode读文件中的内容
* writei : 根据inode写文件

### bmap
函数功能：返回给定inode中第bn个数据块的地址，如果该数据块不存在则创建一个
函数参数：struct inode *ip, uint bn
返回值：uint
函数实现：
* bn < NDIRECT, 代表该数据块是直接盘块，其地址在addr数组中。于是检查addr[bn]是否为0，如果不为零表示该地址是第bn个直接盘块的地址，函数成功返回;否则表示第bn个地址还没有对应的数据块，则使用balloc申请一个数据块，并设置addr[bn]，函数成功返回。
* bn > NDIRECT && bn < NDIRECT + NINDIRECT, 代表bn的地址在间接块中。首先判断间接块是否存在，如果不存在就创建一个间接块。然后将间接块读入内存，判断bn所对应的数据块是否存在，如果不存在则创建，否则返回。
* bn > NDIRECT + NINDIRECT, 代表bn超出了inode所能够拥有的最大数据块数目，报错

### readi
函数功能：根据文件inode指针**将该文件的内容**读到目的地址。注意：不是读inode的内容，而是读文件的实际内容。
函数参数：
* struct inode *ip：待读取文件的inode指针
* char *dst ：目的地址
* uint off： 待读取第一个字节在文件内的偏移
* uint n： 读取的总字节数

返回值：成功则返回实际读到字节数n，如果失败返回-1
函数实现：
* 判断待读取内容是否合法。如果起使位置超过文件长度或者读取总字节数<0，则表示off和n非法，返回-1。如果off + n 超过文件长度，则读从off开始到文件结束部分的内容。
* 如果[off, off+n]位于一个数据块中，则将该数据块读入内存，然后把[off, off+n]copy到dst处即可
* 如果[off ,off+n]跨越多个数据块，则每次读一个数据块，进行多次。
![QtyxAK.png](https://s2.ax1x.com/2019/12/07/QtyxAK.png)
### writei
函数功能：根据文件inode指针将内存中数据复制到文件指定位置。  
函数实现：类似readi，不再赘述
***
## Directory
目录文件和普通文件非常相似，所不同的是目录文件的文件类型为T_DIR，目录文件数据块中存放的是一个个目录项。目录项所对应的数据结构是dirent。每个目录项包括一个文件的文件名name和它的inode的编号inum。
```c
struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```
与目录相关的函数主要有两个：
* dirlookup ：在目录中查找某个文件，如果找到则返回其对应的inode指针；否则返回0
* dirlink : 在目录中添加一个目录项
***
## Pathname
Pathname层主要是在目录层的基础上实现对文件路径名的解析。  
主要包括以下几个函数：
* skipelem ：解析出文件路径中的一层，并返回剩余的pathname  
* namei ：返回给定文件(pathname)的inode指针  
* nameiparent ：返回给定文件所在目录的inode指针  
因为namei和nameiparent都是基于namex这个函数来实现的，所以下面重点分析一下namex。 
### namex
函数功能：返回指定文件或该文件所在目录的inode指针
函数参数：
* char *path ：文件名(如果以/开头，表示绝对路径名，否则表示相对路径名)
* int nameiparent ：标志，如果为1，则解析到父目录即返回
* char *name  ：如果nameiparent为1，解析到父目录并将最后一个文件名复制到name中
返回值：该文件或其所在目录文件对应的inode指针  

函数实现：路径名的解析实际上是一个递归的过程，在这里使用while循环对文件路径进行解析。每一次循环解析该路径上的一个目录。
***
## File descriptor
unix系统中比较重要的一个思想是一切皆文件。大部分资源在unix中都被表示为文件，比如设备、管道以及文件本身。file descriptor层的作用就是实现这种统一。  
在xv6中使用结构体file来代表一个打开文件，每次使用open函数打开一个文件时都会创建一个新的file结构体。
```c
struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE } type;
  int ref; // reference count
  char readable;
  char writable;
  struct pipe *pipe;
  struct inode *ip;
  uint off;
};
```
每个进程有一个自己的打开文件表，在pcb中的ofile数组。从这里可以看出，每个进程最多打开文件个数为NOFILE。
```c
struct proc {
  uint sz;                     // Size of process memory (bytes)
  pde_t* pgdir;                // Page table
  char *kstack;                // Bottom of kernel stack for this process
  enum procstate state;        // Process state
  volatile int pid;            // Process ID
  struct proc *parent;         // Parent process
  struct trapframe *tf;        // Trap frame for current syscall
  struct context *context;     // swtch() here to run process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```
同时，在系统中维护一张系统打开文件表，记录当前系统中所有的打开文件。从这里也可以看出，整个系统最多支持NFILE个文件。
```c
struct {
  struct spinlock lock;
  struct file file[NFILE];
} ftable;
```
与文件相关的函数主要有：
* fileinit ：初始化系统打开文件表ftable
* filealloc ：申请一个file结构体
* filedup ：增加一个文件的引用数
* fileclose ：关闭一个文件(引用数减1，当引用数为0时才关闭)
* filestat ：取一个文件的元数据
* fileread ：读文件
* filewrite ：写文件  

下面重点分析fileclose、fileread、filewrite。
### fileclose
函数功能：关闭一个文件。实际上是将文件的引用数减1，当文件引用数为0时才真正关闭文件。
函数参数：struct file *f
函数实现：判断f的引用数，如果小于1则代表文件已经关闭了，报panic。如果大于1，则将引用数减1，然后直接返回。如果引用数等于1，则回收该文件结构体，并根据文件的类型执行相应的关闭操作。
* 如果文件f是管道文件，则执行管道文件关闭函数piepclose
* 如果文件f类型是FD_INODE，则代表是普通文件，执行一次transaction将日志块写回到相应的数据块。
### fileread
函数功能：从文件读写指针处开始，读n个字节到内存指定地址处
函数实现：首先判断文件是否可读，如果不可读则直接返回-1。然后根据文件的类型执行相应的读操作。如果是管道文件就调用piepread函数，否则，调用readi进行读操作。  
疑问：如果是设备呢？
### filewrite
函数功能：将内存中指定地址处的n个字节数据写到文件读写指针处  
函数实现：读文件和写文件不同之处在于，读文件不需要用到log层，而写文件需要先写到日志块上。所以当需要写的数据很多时，受限于日志块的个数，一次transaction写的字节数是有限的，需要拆分为多个trans。while这个循环就是将要写的很多数据拆成多个trans。
***
## System calls
* sys_dup ：将指定文件的引用数加1
* sys_read ：读文件
* sys_write ：写文件
* sys_close ：关闭文件
* sys_fstat ：获得文件的元数据
* sys_link ：设置链接
* sys_unlink ：取消链接
* sys_open ：打开文件
* sys_mkdir ：创建目录
* sys_mknod ：创建外设
* sys_chdir ：切换进程的cwd(当前目录)
* sys_exec ：执行文件
* sys_pipe ：创建管道
***