# xv6 Startup + Organization

## List of files

```
kernel / bio.c                      user / initcode.S  √
         buf.h                             sh.c
         console.c                         cat.c
         defs/h                            echo.c
         elf.h                             ...
         entry.S  √
         exec.c
         fcntl.h
         file.c
         file.h
         fs.c
         fs.h
         kalloc.c
         kernel.ld
         kernelvec.S  √
         log.c
      √  main.c
         memlayout.h
         param.h
         pipe.c
         plic.c
         printf.c
         proc.c
         proc.h
         ramdisk.c
         riscv.h
         sleeplock.c
         sleeplock.h
         spinlock.c
         spinlock.h
      √  start.c
         stat.h
         string.c
         swtch.S  √
         syscall.c
         syscall.h
         sysfile.c
         sysproc.c
         trampoline.S  √
         trap.c
         types.h
         uart.c
         virtio_disk.c
         virtio.h
         vm.c
```

### xv6 Organization

![](./images/xv6-organization.drawio.svg)

### xv6 startup

![](./images/xv6-startup.drawio.svg)

在 `entry.S` 中，每个 core 设置寄存器 `sp` 和 `tp`。因为每个 core 都有自己独立的栈，所以 `sp` 寄存器存放其所在的 core 的栈顶位置。每个 core 都有自己独立的编号，`tp` 存放的是 core number，即 core 的编号，这个寄存器的值应该保持不变（当然切换到 user 态后可能会覆写 `tp` 的值，但是通过上下文切换，我们可以保证在 kernel  态时，`tp` 存放的值就是 core number）。

## main.c

```c
#include "types.h"
#include "param.h"
#include "memlayout.h"
#include "riscv.h"
#include "defs.h"

volatile static int started = 0;

// start() jumps here in supervisor mode on all CPUs.
void
main()
{
  if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf("\n");
    printf("xv6 kernel is booting\n");
    printf("\n");
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
    kvminithart();   // turn on paging
    procinit();      // process table
    trapinit();      // trap vectors
    trapinithart();  // install kernel trap vector
    plicinit();      // set up interrupt controller
    plicinithart();  // ask PLIC for device interrupts
    binit();         // buffer cache
    iinit();         // inode table
    fileinit();      // file table
    virtio_disk_init(); // emulated hard disk
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf("hart %d starting\n", cpuid());
    kvminithart();    // turn on paging
    trapinithart();   // install kernel trap vector
    plicinithart();   // ask PLIC for device interrupts
  }

  scheduler();        
}
```

因为 xv6 是 multi-core，所以每个 core 都会执行到 main.c 的第一句。而我们只需要一个 core 来进行初始化操作，所以我们通过以下逻辑来实现 core 0 被赋予实现大部分初始化的职责：

```c
volatile static int started = 0; 

void
main()
{
  if (cpuid() == 0) {   // core 0
    ...                 // some init issues
    userinit();         // first user process
    __sync_synchronize();
    started = 1;
  } else {              // other cores
    while (started == 0)
      ;
    __sync_synchronize();
    ...                 // other cores' init
  }

  scheduler();          // switch to exec user program
}
```

- `volatile`：用于指示这个变量会用于多核的同步。
- `__sync_synchronize();`：编译器优化时会对指令进行重排序，这条语句的作用是，指示编译器在优化时要保证，在这条语句后面的指令开始执行之前，必须把这条语句前面的指令全部执行完成。这样可以达到我们预期的同步。
- `userinit()`：初始化用户进程，只需要 core 0 调用即可，因为只有一个零号进程。
- `scheduler()`：最后每个 core 都调用进程调度器，进行进程的调度、执行。

## types.h

```c
typedef unsigned int   uint;
typedef unsigned short ushort;
typedef unsigned char  uchar;

typedef unsigned char uint8;
typedef unsigned short uint16;
typedef unsigned int  uint32;
typedef unsigned long uint64;

typedef uint64 pde_t;
```

声明一些类型别名，注意 RISC-V (RV64) 的类型位宽定义如下：

### RISC-V (RV64)

| type | bits | bytes |
| :--: | :--: | :---: |
| char |   8  |   1   |
| short|  16  |   2   |
| int  |  32  |   4   |
| long |  64  |   8   |

**8 bytes 的`long` 常用于 Addresses 和 Pointers。**

## param.h

```c
#define NPROC        64  // maximum number of processes
#define NCPU          8  // maximum number of CPUs
#define NOFILE       16  // open files per process
#define NFILE       100  // open files per system
#define NINODE       50  // maximum number of active i-nodes
#define NDEV         10  // maximum major device number
#define ROOTDEV       1  // device number of file system root disk
#define MAXARG       32  // max exec arguments
#define MAXOPBLOCKS  10  // max # of blocks any FS op writes
#define LOGSIZE      (MAXOPBLOCKS*3)  // max data blocks in on-disk log
#define NBUF         (MAXOPBLOCKS*3)  // size of disk block cache
#define FSSIZE       2000  // size of file system in blocks
#define MAXPATH      128   // maximum file path name
```

定义一些 xv6 中的硬编码的常量。

## defs.h

```c
struct buf;
struct context;
struct file;
struct inode;
struct pipe;
struct proc;
struct spinlock;
struct sleeplock;
struct stat;
struct superblock;

// bio.c
void            binit(void);
struct buf*     bread(uint, uint);
void            brelse(struct buf*);
void            bwrite(struct buf*);
void            bpin(struct buf*);
void            bunpin(struct buf*);

// console.c
void            consoleinit(void);
void            consoleintr(int);
void            consputc(int);

...

// number of elements in fixed-size array
#define NELEM(x) (sizeof(x)/sizeof((x)[0]))
```

声明各个源文件中的结构体和函数原型的头文件，用于编译链接。

**最后的 `NELEN` 宏，其功能为计算数组的长度，是一个非常有用的功能。**