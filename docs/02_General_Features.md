# xv6 Features

## "SMP": Shared Memory Multi-Processor

- CPU = core = **HART**
- Main Memory (RAM) is Shared **(All caching issues are ignored.)**
- 128 Mbytes (Fixed with `#define` in kernel)

## Devices

- UART (Serial Communication: `Tx <==> Rx`)
- Disk
- Timer Interrupts
- PLIC: Platform Level Interrupt Controller / 用来给 core 转发中断的平台级别的控制器
- CLINT: Core Local Interrupt Controller / 每个 core 自己的中断控制器

## Memory Management

- Page Size = 4096 bytes (`#define PGSIZE` in kernel)
- Single Free List
- No variable-sized allocation
- No `malloc`
- Page Tables
  - Three Levels
  - One table per process + One table for the kernel (内核页表由各个 core 共享)
  - Pages Marked `R/W/X/U/V` **(Read / Write / Exec / User Mode / Valid)**

## Scheduler

- Round-Robin / 这是一个近似于 RR 的调度器，因为多核的存在，使情况变得复杂了
- Size of timeslice is fixed. **(1,000,000 cycles)**
- All cores share one "Ready Queue".
- Next timeslice may be on a different core.

## Boot Sequence

- QEMU
  - Loads Kernel code at fixed address `0x8000_0000`.
- No bootloader / bootblock / BIOS

## Locking

- Spin Locks / 0 表示锁空闲，1 表示 锁被占有
- `sleep()`, `wakeup()`
- 还有一种方法可以对数据实现上锁操作，即关闭中断。但这只对单核的情况有效，因为多核状态下，其它核仍然可以修改内存的数据。

## "PARAM.H"

- Fixed Limits
  - **# of processes**
  - **# of open files**
  - **etc...**
- Several Arrays / 在内核使用数组比使用链表更加高效
  - **"kill (pid) => Linear search of process array**

## User Address Space

```
 MAXVA -----------
       | 1 Page  | <-- "Trampoline" Page (R/X/~U)
       -----------
       | 1 Page  | <-- Trap Frame (R/W/~U)
       -----------
       | / / / / |
       | / / / / |
       | / / / / | (~U)
       | / / / / |
       | / / / / |
   brk -----------
       |    ^    |
       |    |    |
       |    |    | (R/W)
       |  Heap   |
       |         |
       -----------
       | 1 Page  | <-- Stack (R/W)
       -----------
       | 1 Page  | <-- Guard Page (~U)
       -----------
       |  Data   |
       |   and   | <-- Load from ELF executable file (R/W/X)
       |  Code   |
     0 -----------
```

- **“~U”** means "Not Valid when in User Mode."
- Arguments (`argc`, `argv`) will be placed on the stack before the program begins execution.
- 当发生异常/中断时，会跳转到 `"Trampoline" Page` 里执行相应的指令，其主要功能为将该进程的状态（即寄存器的状态），存入到 `Trap Frame` 里。

## RISC-V Vitual Addresses

- Multiple Schemes
  - Sv32 (2-level page table)
  - Sv39 (3-level page table)
  - Sv48 (4-level page table)
- xv6 uses **"Sv39"** architecture
- Virt. Addr. Size
  - 39 bits
  - 2^39 = 512 Gbytes = `0x80_0000_0000`

```
                                  |-------------------39 bits---------------------|
64|       56|       48|       40| |     32|       24|       16|        8|        0|
  |----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
  |         |         |         |  | |    |         |         |         |         |
                                   |------------------38 bits---------------------|
```

- xv6 uses only 38 bits for virtual addresses.
  - 2^38 = 256 Gbytes = `0x40_0000_0000` = `MAXVA`
  - User Space: `0 ... 0x3F_FFFF_FFFF`