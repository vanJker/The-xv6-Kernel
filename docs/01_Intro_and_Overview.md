# The xv6 Kernel

- 类 UNIX 的操作系统
- MIT 开发的用于教学目的的操作系统
- xv6 的实现
  - x86, RISC-V (RISC-V 版本的 xv6 是一个 64 位的 OS)
  - Multicore / 多核
  - 大约 6000 行代码 (C 和 汇编代码, 其中大约有 300 行的汇编代码)

## xv6 的特性

- Process / 进程
- Virtual Address Spaces / 虚拟空间, Page Tables / 页表
- Files / 文件, Directories / 目录
- Pipes / 管道
- Multi-tasking / 多任务, Time-slicing / 时间切片
- 21 System Calls / 实现了 21 个系统调用

## xv6 的用户程序

| User Programs |
| :-----------: |
| sh |
| cat |
| echo |
| grep |
| kill |
| ln |
| ls |
| mkdir |
| rm |
| wc |

## xv6 的缺陷

缺乏真正操作系统的复杂性：

- User IDs / 用户 ID, Login / 登陆功能
- File Protection / 文件保护
- "Mount"able File Systems / 可挂载的文件系统
- Paging to disk / 在磁盘中分页
- Sockets / 套接字, support for networks / 网络支持
- Interprocess Communication / 进程间通信
- Only 2 Device Drivers / 只有 2 个设备驱动
- User code / Apps / 只有有限的用户程序

## xv6 的系统调用

```c
// system calls
int fork(void);                         // 创建新的进程
int exit(int) __attribute__((noreturn));// 退出进程
int wait(int*);                         // 等待进程结束
int pipe(int*);                         // 创建管道
int write(int, const void*, int);       // 写文件
int read(int, void*, int);              // 读文件
int close(int);                         // 关闭文件
int kill(int);                          // 终止子进程
int exec(char*, char**);                // 执行程序
int open(const char*, int);             // 打开文件
int mknod(const char*, short, short);   // 创建设备文件
int unlink(const char*);                // 取消硬链接
int fstat(int fd, struct stat*);        // 获取文件信息
int link(const char*, const char *);    // 创建硬链接
int mkdir(const char*);                 // 创建目录
int chdir(const char*);                 // 改变当前目录
int dup(int);                           // 复制文件描述符
int getpid(void);                       // 获取当前进程标识符
char* sbrk(int);                        // 申请更多内存给堆
int sleep(int);                         // 当前进程休眠一段时间
int uptime(void);                       // 系统到当前一共运行了多长时间
```