## lab2
### System call tracing
**该实验本质上是修改系统调用的syscall()函数，从而在每次调用时输出进程pid和名称等信息，前面的大量工作是为了配置trace命令运行的环境以及针对这项命令修改proc结构体和fork函数**
该任务主要是为了记录系统调用的过程，但是由于环境都未配置，因此前期操作繁琐  
```c
//trace.c文件，已给出
#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  int i;
  char *nargv[MAXARG];

  //验证掩码为数字
  if(argc < 3 || (argv[1][0] < '0' || argv[1][0] > '9')){
    fprintf(2, "Usage: %s mask command\n", argv[0]);
    exit(1);
  }
  //验证掩码为非负数
  if (trace(atoi(argv[1])) < 0) {
    fprintf(2, "%s: trace failed\n", argv[0]);
    exit(1);
  }
  
  for(i = 2; i < argc && i < MAXARG; i++){
    nargv[i-2] = argv[i];
  }
  exec(nargv[0], nargv);    //执行后续需要trace的命令
  exit(0);
}
```
配置过程为  
- 在Makefile中增加$U/_trace项
- 系统调用的用户空间存根还不存在：向user/user.h添加系统调用的原型(user.h中包含system calls函数的声明)
```c
//截取部分声明如下，trace函数的参数为mask,故为int类型
int sleep(int);
int uptime(void);
int trace(int);
```
- 在 user/usys.pl 中增加entry("trace"); (perl脚本) 
- 在 kernel/syscall.h 中为trace命令定义一个对应数字 #define SYS_trace  22
- 在 kernel/sysproc.c 中新定义一个sys_trace()函数用于将trace命令的参数传递到proc结构体的mask参数中
```c
uint64
sys_trace(void)
{
    argint(0, &(myproc()->mask)); //argint函数在syscalls.c中定义，用于参数传递
    return 0;
}

//argint
// Fetch the nth 32-bit system call argument.
int
argint(int n, int *ip)
{
  *ip = argraw(n);
  return 0;
}

//argraw
//这里ax分别存储什么数据还不是很清楚，在本实验中用到的是a0，存放的是trace命令的参数
static uint64
argraw(int n)
{
  struct proc *p = myproc();
  switch (n) {
  case 0:
    return p->trapframe->a0;
  case 1:
    return p->trapframe->a1;
  case 2:
    return p->trapframe->a2;
  case 3:
    return p->trapframe->a3;
  case 4:
    return p->trapframe->a4;
  case 5:
    return p->trapframe->a5;
  }
  panic("argraw");
  return -1;
}
```
- 修改kernel/proc.c中的fork()函数，fork创建新进程时同时复制mask(np->mask=p->mask;)
```c
// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;

  np->parent = p;

  // copy saved user registers.
  *(np->trapframe) = *(p->trapframe);

  // Cause fork to return 0 in the child.
  np->trapframe->a0 = 0;

  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  safestrcpy(np->name, p->name, sizeof(p->name));

  np->mask=p->mask; //Copy Mask Number (revised)

  pid = np->pid;

  np->state = RUNNABLE;

  release(&np->lock);

  return pid;
}

```
- 修改kernel/syscall.c中的syscall()函数，以输出所需内容
```c
//注意补充前面的声明
//补充系统调用名称数组
//ps：这个初始化方式没看懂需要继续研究
static char *syscalls_name[] = {
[SYS_fork]    "fork",
[SYS_exit]    "exit",
...
[SYS_trace]   "trace",
};
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    if(((1<<num)&(myproc()->mask))!=0)
      printf("%d: syscall %s -> %d\n", myproc()->pid, syscalls_name[num], p->trapframe->a0);
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

### Sysinfo
此任务的主要目标是写一个获取系统信息的调用，读取系统的空余内存和进程数  
1. 增加一个sysinfotest命令
    - 照例在Makefile中增加$U/_sysinfotest\项
    - 并在user.h中增加sysinfotest的函数声明和相关结构体声明
2. sysinfo系统调用
    - 在kernel/syscall.h中为SYS_sysinfo定义对应数字
    - 在kernel/syscall.c中的syscalls指针数组中增加sys_sysinfo项
    - 在kernel/sysproc.c中增加sys_sysinfo函数的实现
3. sysinfo的实现
主要包括剩余内存读取，进程数读取以及结构体复制三部分
```c
//剩余内存读取
//位于kalloc.c中，该文件用于管理内存
void
freeSpace(uint64 *space){
  *space = 0;
  struct run *p = kmem.freelist;
  //Iterate Freelist
  acquire(&kmem.lock); //锁
  while(p){ //遍历freelist并统计所有剩余空间
    *space+=PGSIZE;
    p=p->next;
  }
  release(&kmem.lock);
}

//进程数读取
//位于proc.c中，该文件用于进程管理
void
procCnt(uint64 *cnt){
  *cnt=0;
  struct proc *p;
  for (p = proc; p < &proc[NPROC]; p++) {
    if (p->state != UNUSED)
      (*cnt)++;
  }
}

//将结构体从kernel复制到user中
uint64
sys_sysinfo(void)
{
  //kernel/kalloc.c  freespase()
  //kernel/proc.c    procCnt()
  struct sysinfo info;
  freeSpace(&info.freemem);
  procCnt(&info.nproc);
  //copyout struct from kernel to user
  uint64 virtualAddr;
  argaddr(0,&virtualAddr);  //读取虚拟地址，并使用copyout函数复制结构体
  if (copyout(myproc()->pagetable, virtualAddr, (char *)&info, sizeof info) < 0)
    return -1;
  return 0;
}

//ps:copyout位于vm.c中
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
```

