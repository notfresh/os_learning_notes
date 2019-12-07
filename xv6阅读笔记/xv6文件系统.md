# 文件系统
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
* 对buf进行操作
* log_write()将buf暂时写到日志块
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
* readi : 读inode
* writei : 写inode

### bmap
函数功能：返回给定inode中第bn个数据块的地址，如果该数据块不存在则创建一个
函数参数：struct inode *ip, uint bn
返回值：uint
函数实现：
* bn < NDIRECT, 代表该数据块是直接盘块，其地址在addr数组中。于是检查addr[bn]是否为0，如果不为零表示该地址是第bn个直接盘块的地址，函数成功返回;否则表示第bn个地址还没有对应的数据块，则使用balloc申请一个数据块，并设置addr[bn]，函数成功返回。
* bn > NDIRECT && bn < NDIRECT + NINDIRECT, 代表bn的地址在间接块中。首先判断间接块是否存在，如果不存在就创建一个间接块。然后将间接块读入内存，判断bn所对应的数据块是否存在，如果不存在则创建，否则返回。
* bn > NDIRECT + NINDIRECT, 代表bn超出了inode所能够拥有的最大数据块数目，报错

### readi

### writei


## Directory

## Pathname

## File descriptor
