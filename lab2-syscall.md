# lab2: syscall
[源码链接](https://github.com/babyseal1121/6.S081.git)
##实验内容

## trace

本实验主要是实现一个追踪系统调用的函数，那么首先根据提示定义`trace`系统调用，并修复编译错误。
####实验要求
添加一个系统调用跟踪功能，该功能可能会在以后调试实验有帮助。将创建一个新的跟踪系统调用来控制跟踪。它应该采用一个参数，一个整数“掩码”，其位指定要跟踪的系统调用。
####实验步骤

（1）最先要做的必然是添加$U/_trace\到Makefile中的UPROGS变量里

（2）proc结构体里的name是整个线程的名字，不是函数调用的函数名称，所以我们不能用p->name，而要自己定义一个数组，在/kernel/syscall.c文件添加数组：
```c

	char *syscalls[] = {
	[SYS_fork]    "fork",
	[SYS_exit]    "exit",
	[SYS_wait]    "wait",
	[SYS_pipe]    "pipe",
	[SYS_read]    "read",
	[SYS_kill]    "kill",
	[SYS_exec]    "exec",
	[SYS_fstat]   "stat",
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


（3）把trace这个系统调用加入到内核中声明，
	/usre/user.h文件加入int trace(int);
/user/usys.pl文件加入entry("trace");

（4）在kernel/syscall.c加上extern uint64 sys_trace(void);

（5）在函数指针数组*syscalls[]加上：[SYS_trace]   sys_trace,

（6）在kerlnel/sysproc.c加上sys_trace的定义实现，把传进来的参数给到现有进程的mask.实现如下：
```c

	uint64
	sys_trace(void)
	{
  		int mask;
  		if(argint(0, &mask) < 0)
    	return -1;
  
  		myproc()->mask = mask;
  		return 0;
	}

```

（7）kernel/syscall.h中添加#define SYS_trace22

（8）要修改kernel/proc.h中的proc结构（记录当前进程信息），给proc添加一个mask值用来识别system number，即添加 int mask;

（9）修改kernel/proc.c中的fork函数，添加子进程复制父进程mask的功能（ np->mask = p->mask;）

（10）输入命令 make qemu进行结果检验

（11）检验完后Ctrl+a -x退出qemu，输入./grade-lab-syscall trace进行测试。

## sysinfo
####实验要求
添加一个系统调用sysinfo，它收集有关正在运行的系统的信息。系统调用有一个参数：指向struct sysinfo的指针 （参见kernel/sysinfo.h）。内核应填写此结构的字段：freemem字段应设置为可用内存的字节数，nproc 字段应设置为状态不是UNUSED的进程数。
3.2.2实验步骤
####实验步骤

（1）获取状态不是UNUSED的进程数，在kernel/proc.c中加入：
```c
	
	int
	proc_size()
	{
	  int i;
	  int n = 0;
	  for (i = 0; i < NPROC; i++)
	  {
	    if (proc[i].state != UNUSED) n++;
	  }
	  return n;
	}
```

（2）获取可用内存字节数，kernel/kalloc.c中加入
```c

	uint64 
	freememory()
	{
	  struct run* p = kmem.freelist;
	  uint64 num = 0;
	  while (p)
	  {
	    num ++;
	    p = p->next;
	  }
	  return num * PGSIZE;
	}
```

（3）sysinfo系统调用的实现，kernel/sysproc.c中加入
```c

	uint64
	sys_sysinfo(void)
	{
	  struct sysinfo info;
	  uint64 addr;
	  // 获取用户态传入的sysinfo结构体
	  if (argaddr(0, &addr) < 0) 
	    return -1;
	  struct proc* p = myproc();
	  info.freemem = freememory();
	  info.nproc = proc_size();
	  // 将内核态中的info复制到用户态
	  if (copyout(p->pagetable, addr, (char*)&info, sizeof(info)) < 0)
	    return -1;
	  return 0;
	}

```

（4）与上个实验一样，需要一些声明和定义。
```c

	Makefile 文件中添加配置，在 UPROGS 项中最后添加 $U/_sysinfotest\
	在kernel/syscall.h中宏定义 #define SYS_sysinfo 23
	在user/usys.pl中新增一个entry("sysinfo");
	在user/user.h中新增sysinfo结构体说明：struct sysinfo;
	在user/user.h中新增sysinfo函数说明：int sysinfo(struct sysinfo*);
	kernel/syscall.c中新增sys_trace函数定义：extern uint64 sys_sysinfo(void);
	kernel/syscall.c中函数指针数组新增：
	[SYS_sysinfo]   sys_sysinfo,
	[SYS_sysinfo]   "sysinfo",
	kernel/defs.h做出函数声明：
	int 		 proc_size(void);
	uint64          freememory(void);
	kernel/sysproc.c加入sysinfo.h的header头：#include "sysinfo.h"
```


（5）输入命令 make qemu进行结果检验

（6）检验完后Ctrl+a -x退出qemu，输入./grade-lab-syscall sysinfo进行测试
##心得体会与总结
####trace
1. 在这个实验里，我们需要让内核输出每个mask变量指定的系统函数的调用情况，格式为：<pid>: syscall <syscall_name> -> <return_value>pid是进程序号， syscallname是函数名称，returnvalue是该系统调用返回值，并且要求各个进程的输出是独立的，不相互干扰，所以我们要在/kernel/proc.h文件的proc结构体中加入一个新的变量，让每个进程都有一个自己的mask。
2. 主要的实现就是在/usr/syscall.c文件的syscall函数，阅读函数原型可知，p->trapframe->a0 = syscalls[num]();这一行就是调用了系统调用命令，并且把返回值保存在了a0寄存器中（RISCV的C规范是把返回值放在a0中，指令码放置在a7寄存器中)，所以我们只要在调用系统调用时判断是不是mask规定的输出函数，如果是就输出
3. 有几个点需要考虑，一个是mask是按位判断的，第二个是proc结构体里的name是整个线程的名字，不是函数调用的函数名称，所以我们不能用p->name，而要自己定义一个数组。
4. 刚开始做实验时出现的一些错误主要在于不清楚添加一个新的功能需要在哪些文件中增加语句，比如在fork（）函数中增加父进程的mask传给子进程，为了生成进入中断的汇编文件，需要在 user/usys.pl 添加进入内核态的入口函数的声明：entry("trace");，以便使用 ecall 中断指令进入内核态等等。
5. p->mask & (1 << num)语句含义一开始不能理解
6. 学习过程中帮助理解的图片：
![](https://img2020.cnblogs.com/blog/853921/202201/853921-20220113171702431-1887170168.png)
####sysinfo
1. 在/kernel/proc.c文件中一开始就定义了一个数组，struct proc proc[NPROC];
这个数组就保存着所有的进程，所以只要遍历这个数组判断状态就好了，判断状态就是在proc结构体中的enum procstate state;
2. 可用空间判断则是在/kernel/kalloc.c文件中，这里它定义了一个链表，每个链表都指向上一个可用空间，这个kmem就是一个保存最后链表的变量。做了这个实验才慢慢理清了一些我之前很模糊的点，在操作系统课程里面我学到了关于用户态和内核态的转换，比如之前做实验1时sleep的部分，虽然实现比较简单，就直接系统调用就好了，原理么大伙都说用户态到内核态然后调用kernel里的sys_sleep，偏偏对这个转换过程总是比较模糊，虽然这个实验也就给了一个usys.pl代码，但确实给了我一个总体清晰的思路
####实验结果
![](https://img-blog.csdnimg.cn/4a50848f8ae045fb8585af65425e3752.png#pic_center)