---

---
### 为何没lab1
lab1 我 fork 的是官方仓库 git://g.csail.mit.edu/xv6-labs-2020
但是我做完 lab1 才发现，不知道为什么没有 lab2 的 syscall 分支
然后我 fork https://github.com/CalvinHaynes/MIT6.S081-2020-labs 之后就把 lab1 全删了
没备份 QAQ

---
# System call tracing
### 步骤
> **在**_kernel/sysproc.c_**中添加一个 `sys_trace()` 函数，它通过将参数保存到 `proc` 结构体（请参见**_kernel/proc.h_**）里的一个新变量中来实现新的系统调用。从用户空间检索系统调用参数的函数在**_kernel/syscall.c_**中，您可以在**_kernel/sysproc.c_**中看到它们的使用示例。**
```c
// kernel/sysproc.c
uint64
sys_trace(void){
  argint(0, &(myproc()->trace_mask));
  return 0;
}

// kernel/proc.h
struct proc {
  // ...
  
  // these are private to the process, so p->lock need not be held.
  // ...
  int trace_mask;              // trace mask
};
```

> **修改 `fork()`（请参阅**_kernel/proc.c_**）将跟踪掩码从父进程复制到子进程。**
```c
// kernel/proc.c
fork(void){
  // ...
  np->trace_mask = p->trace_mask;
  return pid;
}
```

> **修改**_kernel/syscall.c_**中的 `syscall()` 函数以打印跟踪输出。您将需要添加一个系统调用名称数组以建立索引。**
```c
// kernel/syscall.c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();

    if((1 << num) & p->trace_mask){
      printf("%d: syscall %s -> %d\n", p->pid, syscalls_name[num], p->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n", p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```
其中：
+ 当系统调用接口函数返回时，`syscall` 将其返回值记录在 `p->trapframe->a0` 中。
+ 用户代码将 `exec` 需要的参数放在寄存器 `a0` 和 `a1` 中，并将系统调用号放在 `a7` 中。系统调用号与 `syscalls` 数组中的条目相匹配，`syscalls` 数组是一个函数指针表（**_kernel/syscall.c_**:108）。

> **将系统调用的原型添加到** _user/user.h_
> **将存根添加到**_user/usys.pl_
> **将系统调用编号添加到** _kernel/syscall.h_
> **在**_Makefile_**的**_UPROGS_**中添加 `$U/_trace`**

```c
// user/user.h
int trace(int);

// user/usys.pl
entry("trace");

// kernel/syscall.h
#define SYS_trace  22

// kernel/syscall.c
extern uint64 sys_trace(void);

static uint64 (*syscalls[])(void) = {
// ...
[SYS_trace]   sys_trace,
};

static char *syscalls_name[] = {
[SYS_fork]    "fork",
[SYS_exit]    "exit",
[SYS_wait]    "wait",
[SYS_pipe]    "pipe",
[SYS_read]    "read",
[SYS_kill]    "kill",
[SYS_exec]    "exec",
[SYS_fstat]   "fstat",
[SYS_chdir]   "chdir",
[SYS_dup]     "dup",
[SYS_getpid]  "getpid",
[SYS_sbrk]    "sbrk",
[SYS_sleep]   "sleep",
[SYS_uptime]  "uptime",
[SYS_open]    "open",
[SYS_write]   "write",
[SYS_mknod]   "mknod",
[SYS_unlink]  "unlink",
[SYS_link]    "link",
[SYS_mkdir]   "mkdir",
[SYS_close]   "close",
[SYS_trace]   "trace",
};
```


### 结果
![image.png](https://s2.loli.net/2025/06/08/JsXe9I62MkoHvS1.png)


# Sysinfo
### 步骤

> **在_user/user. h_中声明 `struct sysinfo` 的存在和 `sysinfo()` 的原型

> **添加 `sysinfo` 系统调用**
> **`sysinfo` 需要将一个 `struct sysinfo` 复制回用户空间**
>**请参阅 `sys_fstat()` (**_kernel/sysfile.c_**)和 `filestat()` (**_kernel/file.c_**)以获取如何使用 `copyout()` 执行此操作的示例。**
```c
// kernel/sysproc.c
uint64
sys_sysinfo(void)
{
  struct sysinfo info;
  kgetfreebytes(&info.freemem);
  getprocnum(&info.nproc);

  // 获取虚拟地址
  uint64 dstaddr;
  argaddr(0, &dstaddr);

  // 从内核空间拷贝数据到用户空间
  if (copyout(myproc()->pagetable, dstaddr, (char *)&info, sizeof info) < 0)
    return -1;

  return 0;
}
```

> **要获取空闲内存量，请在**_kernel/kalloc.c_**中添加一个函数
> 要获取进程数，请在**_kernel/proc.c_**中添加一个函数
> 将这两个函数的声明添加进**_kernel/defs. h_
```c
// kernel/kalloc.c
void kgetfreebytes(uint64 *dst)
{
  *dst = 0;
  struct run *p = kmem.freelist;
  
  acquire(&kmem.lock);
  while (p) {
    *dst += PGSIZE;
    p = p->next;
  }
  release(&kmem.lock);
}

// kernel/proc.c
void getprocnum(uint64 *dst)
{
  *dst = 0;
  struct proc *p;
  for (p = proc; p < &proc[NPROC]; p++) {
    if (p->state != UNUSED)
      (*dst)++;
  }
}

// kernel/defs.h
// ...
```

> **在**_kernel/sysproc.c_**顶部包含**_sysinfo.h_
```c
#include "sysinfo.h"
```

> **将系统调用的原型添加到** _user/user. h_
> **将存根添加到**_user/usys. pl_
> **将系统调用编号添加到** _kernel/syscall. h_
> **在**_Makefile_**的**_UPROGS_**中添加 `$U/_sysinfo`**

### 结果
![image.png](https://s2.loli.net/2025/06/08/qyjCn2vAzYNWELD.png)

# 总结
##### 系统调用流程 ：
用户程序 → `ecall` → `trap` → ` usertrap ` → ` syscall() ` → ` sys_xxx() ` → 返回用户态
1. **用户程序发起系统调用**  
    用户程序通过执行 `ecall` 指令（在RISC-V 架构中）发起系统调用，并将系统调用号和参数放入寄存器。
    
2. **陷入内核态**  
    `ecall` 指令触发陷入（trap），CPU 切换到内核态，跳转到内核的 trap 处理函数（如 `kernel/trap.c` 的 `usertrap`）。
    
3. **trap 处理函数识别系统调用**  
    `usertrap` 检查 trap 原因，发现是系统调用（通常通过 `scause` 寄存器判断），然后调用 `syscall()` 函数（在 `kernel/syscall.c`）。
    
4. **系统调用分发**  
    `syscall()` 函数根据用户传入的系统调用号（通常在 a7 寄存器），在系统调用表（如 `syscalls[]` 数组）中查找对应的内核实现函数指针。
    
5. **执行具体系统调用函数**  
    通过函数指针调用具体的系统调用实现函数（如 `sys_write`、`sys_exit`、`sys_sysinfo` 等），并传递参数。
    
6. **返回用户态**  
    系统调用函数执行完毕后，将返回值放入寄存器（如 a0），trap 处理函数恢复用户态上下文，返回用户程序继续执行。