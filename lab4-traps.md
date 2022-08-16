## Lab 4: Traps
[源码链接](https://github.com/babyseal1121/6.S081.git)
##实验内容
#### RISC-V assembly (easy)
##### 实验要求
阅读 call.asm 中函数g、f和main的代码，并回答问题

##### 问题答案
```

Q: Which registers contain arguments to functions? For example, which register holds 13 in main's call to printf?

A: a0-a7; a2;

Q: Where is the call to function f in the assembly code for main? Where is the call to g? (Hint: the compiler may inline functions.)

A: There is none. g(x) is inlined within f(x) and f(x) is further inlined into main()

Q: At what address is the function printf located?

A: 0x0000000000000628, main calls it with pc-relative addressing.

Q: What value is in the register ra just after the jalr to printf in main?

A: 0x0000000000000038, next line of assembly right after the jalr

Q: Run the following code.

	unsigned int i = 0x00646c72;
	printf("H%x Wo%s", 57616, &i);      

What is the output?
If the RISC-V were instead big-endian what would you set i to in order to yield the same output?
Would you need to change 57616 to a different value?

A: "He110 World"; 0x726c6400; no, 57616 is 110 in hex regardless of endianness.

Q: In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?

	printf("x=%d y=%d", 3);

A: A random value depending on what codes there are right before the call.Because printf tried to read more arguments than supplied.
The second argument `3` is passed in a1, and the register for the third argument, a2, is not set to any specific value before the
call, and contains whatever there is before the call.

```



#### Backtrace (moderate)
##### 实验要求
实验要求：当调用sys_sleep时打印出函数调用关系的backtrace，即递归地打印每一个函数以及调用这个函数及其父函数的return address
##### 实验步骤
1. gcc将当前执行的函数的frame pointer存储在s0寄存器中，可以在kernel/riscv.h中声明以下函数来获取s0
```c
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```
当前函数的return address位于fp-8的位置，previous frame pointer位于fp-16的位置。

2. 每个kernel stack都是一页，因此可以通过计算PGROUNDDOWN(fp)和PGROUNDUP(fp)来确定当前的fp地址是否还位于这一页内，从而可以是否已经完成了所有嵌套的函数调用的backtrace。

在kernel/printf.c中
```c
void
backtrace(void)
{
  uint64 fp, ra, top, bottom;
  printf("backtrace:\n");
  fp = r_fp();  // frame pointer of currently executing function
  top = PGROUNDUP(fp);
  bottom = PGROUNDDOWN(fp);
  while (fp < top && fp > bottom)
  {
    ra = *(uint64 *)(fp-8);
    printf("%p\n", ra);
    fp = *(uint64 *)(fp-16); // saved previous frame pointer
  }
}
```

#### Alarm (hard)
##### 实验要求
实验要求：添加一个新的system callsigalarm(interval, handler)，interval是一个计时器的tick的间隔大小，handler是指向一个函数的指针，这个函数是当计时器tick到达interval时触发的函数。
在本练习中，您将向xv6添加一项功能，该功能会在使用CPU时间的情况下定期向进程发出警报。 这对于想要限制消耗多少CPU时间的计算密集型进程，或者对于想要进行计算但还希望采取一些定期操作的进程很有用。 更一般而言，您将实现用户级中断/故障处理程序的原始形式。 例如，您可以使用类似的方法来处理应用程序中的页面错误。 您的解决方案是否通过Alarmtest和UserTests是正确的。

您应该添加一个新的sigalarm（interval，handler）系统调用。 如果应用程序调用sigalarm（n，fn），则在程序每消耗n个“ tick” CPU时间之后，内核应导致调用应用程序函数fn。 当fn返回时，应用程序应从中断处恢复。 滴答是xv6中相当随意的时间单位，由硬件计时器产生中断的频率决定。 如果应用程序调用sigalarm（0，0），则内核应停止生成定期警报调用。

您将在xv6存储库中找到一个文件user / alarmtest.c。 将其添加到Makefile。 在您添加了sigalarm和sigreturn系统调用之前，它无法正确编译（请参见下文）。

alarmtest在test0中调用sigalarm（2，periodic），以要求内核每2个滴答强制一次对periodic（）的调用，然后旋转一段时间。 您可以在user / alarmtest.asm中看到alarmtest的汇编代码，这对于调试很方便。 当alarmtest产生这样的输出并且usertests也正确运行时，您的解决方案是正确的：
##### 实验步骤
1.首先修改Makefile使得alarmtest.c能够被编译，在user/user.h中添加函数声明

```c
	
	
	int sigalarm(int ticks, void (*handler)());
	int sigreturn(void);
```

2.在user/user.pl、kernel/syscall.h、kernel/syscall.c等添加sys_sigalarm和sys_sigreturn这两个syscall的注册

3.在kernel/proc.h的proc结构体中添加几个成员。int interval是保存的定时器触发的周期，void (*handler)()是指向handler函数的指针，这两个成员变量在sys_sigalarm中被保存。ticks用来记录从上一次定时器被触发之后到目前的ticks，in_handler用来记录当前是否在handler函数中，用来防止在handler函数的过程中定时被触发再次进入handler函数。下面的所有寄存器都是用来保护和恢复现场的。

```c


	int interval;                // interval of alarm
	  void (*handler)();           // pointer to the handler function
	  int ticks;                   // how many ticks have passed since last call
	  int in_handler;              // to prevent from reentering into handler when in handler
	  uint64 saved_epc;
	  uint64 saved_ra;
	  uint64 saved_sp;
	  uint64 saved_gp;
	  uint64 saved_tp;
	  uint64 saved_t0;
	  uint64 saved_t1; 
	  uint64 saved_t2;
	  uint64 saved_s0;
	  uint64 saved_s1;
	  uint64 saved_s2;
	  uint64 saved_s3;
	  uint64 saved_s4;
	  uint64 saved_s5;
	  uint64 saved_s6;
	  uint64 saved_s7;
	  uint64 saved_s8;
	  uint64 saved_s9;
	  uint64 saved_s10;
	  uint64 saved_s11;
	  uint64 saved_a0;
	  uint64 saved_a1;
	  uint64 saved_a2;
	  uint64 saved_a3;
	  uint64 saved_a4;
	  uint64 saved_a5;
	  uint64 saved_a6;
	  uint64 saved_a7;
	  uint64 saved_t3;
	  uint64 saved_t4;
	  uint64 saved_t5;
	  uint64 saved_t6;
```

4.在kernel/sysfile.c中分别添加uint64 sys_sigreturn(void)和uint64 sys_sigalarm(void)的实现

5.对于sys_sigalarm，需要将从userfunction传入的两个参数分别保存到p结构体相应的成员变量中
```c

	uint64
	sys_sigalarm(void)
	{
	  int ticks;
	  uint64 funaddr; // pointer to function
	  if ((argint(0, &ticks) < 0) || (argaddr(1, &funaddr) < 0)) {
	    return -1;
	  }
	  struct proc *p = myproc();
	  p->interval = ticks;
	  p->handler = (void(*)())funaddr;
	  return 0;
	}
```
6.对于sys_sigreturn，这个函数会在handler函数完成之后被调用，它的主要工作是恢复现场的寄存器，并且将in_handler这个flag置0
```c

	
	uint64
	sys_sigreturn(void)
	{
	  struct proc *p = myproc();
	  p->trapframe->epc = p->saved_epc;
	  p->trapframe->ra = p->saved_ra;
	  p->trapframe->sp = p->saved_sp;
	  p->trapframe->gp = p->saved_gp;
	  p->trapframe->tp = p->saved_tp;
	  p->trapframe->t0 = p->saved_t0;
	  p->trapframe->t1 = p->saved_t1;
	  p->trapframe->t2 = p->saved_t2;
	  p->trapframe->t3 = p->saved_t3;
	  p->trapframe->t4 = p->saved_t4;
	  p->trapframe->t5 = p->saved_t5;
	  p->trapframe->t6 = p->saved_t6;
	  p->trapframe->s0 = p->saved_s0;
	  p->trapframe->s1 = p->saved_s1;
	  p->trapframe->s2 = p->saved_s2;
	  p->trapframe->s3 = p->saved_s3;
	  p->trapframe->s4 = p->saved_s4;
	  p->trapframe->s5 = p->saved_s5;
	  p->trapframe->s6 = p->saved_s6;
	  p->trapframe->s7 = p->saved_s7;
	  p->trapframe->s8 = p->saved_s8;
	  p->trapframe->s9 = p->saved_s9;
	  p->trapframe->s10 = p->saved_s10;
	  p->trapframe->s11 = p->saved_s11;
	  p->trapframe->a0 = p->saved_a0;
	  p->trapframe->a1 = p->saved_a1;
	  p->trapframe->a2 = p->saved_a2;
	  p->trapframe->a3 = p->saved_a3;
	  p->trapframe->a4 = p->saved_a4;
	  p->trapframe->a5 = p->saved_a5;
	  p->trapframe->a6 = p->saved_a6;
	  p->trapframe->a7 = p->saved_a7;
	  p->in_handler = 0;
	  return 0;
	}
```

7.kernel/trap.c中的usertrap()是对陷入定时器中断的处理，需要判断which_dev==2才是定时器中断，当p->ticks到达预设值p->interval时，将调用p->handler，这是通过向p->trapframe->epc加载handler地址实现的，这样当从usertrap()中返回时，pc在恢复现场时会加载p->trapframe->epc，这样就会跳转到handler地址。在跳转到handler之前先要保存p->trapframe的寄存器，因为执行handler函数会导致这些寄存器被覆盖。
```c

	
	syscall();
	  } else if((which_dev = devintr()) != 0){
	    // ok
	    if (which_dev == 2 && p->in_handler == 0) {
	      p->ticks += 1;
	      if ((p->ticks == p->interval) && (p->interval != 0)) {
	        p->in_handler = 1;
	        p->ticks = 0;
	        p->saved_epc = p->trapframe->epc;
	        p->saved_ra = p->trapframe->ra;
	        p->saved_sp = p->trapframe->sp;
	        p->saved_gp = p->trapframe->gp;
	        p->saved_tp = p->trapframe->tp;
	        p->saved_t0 = p->trapframe->t0;
	        p->saved_t1 = p->trapframe->t1;
	        p->saved_t2 = p->trapframe->t2;
	        p->saved_t3 = p->trapframe->t3;
	        p->saved_t4 = p->trapframe->t4;
	        p->saved_t5 = p->trapframe->t5;
	        p->saved_t6 = p->trapframe->t6;
	        p->saved_s0 = p->trapframe->s0;
	        p->saved_s1 = p->trapframe->s1;
	        p->saved_s2 = p->trapframe->s2;
	        p->saved_s3 = p->trapframe->s3;
	        p->saved_s4 = p->trapframe->s4;
	        p->saved_s5 = p->trapframe->s5;
	        p->saved_s6 = p->trapframe->s6;
	        p->saved_s7 = p->trapframe->s7;
	        p->saved_s8 = p->trapframe->s8;
	        p->saved_s9 = p->trapframe->s9;
	        p->saved_s10 = p->trapframe->s10;
	        p->saved_s11 = p->trapframe->s11;
	        p->saved_a0 = p->trapframe->a0;
	        p->saved_a1 = p->trapframe->a1;
	        p->saved_a2 = p->trapframe->a2;
	        p->saved_a3 = p->trapframe->a3;
	        p->saved_a4 = p->trapframe->a4;
	        p->saved_a5 = p->trapframe->a5;
	        p->saved_a6 = p->trapframe->a6;
	        p->saved_a7 = p->trapframe->a7;
	        p->trapframe->epc = (uint64)p->handler;
	      }
	    }
	  } else {
	```
####实验心得及总结
#####前置知识
1. 函数栈帧：x86 使用的函数参数压栈的方式来保存函数参数，xv6 使用寄存器的方式保存参数。无论是x86还是xv6，函数调用时，都需要将返回地址和调用函数（父函数）的栈帧起始地址压入栈中。即被调用函数的栈帧中保存着这两个值。在 xv6 中，fp为当前函数的栈顶指针，sp为栈指针。fp-8 存放返回地址，fp-16 存放原栈帧（调用函数的fp）。
2. 系统调用：首先，当用户调用系统调用的函数时，在进入函数前，会执行 user/usys.S 中相应的汇编指令，指令首先将系统调用的函数码放到a7寄存器内，然后执行 ecall 指令进入内核态。
ecall 指令是 cpu 指令，该指令只做三件事情。
（1）首先将cpu的状态由用户态（user mode）切换为内核态（supervisor mode）；
（2）然后将程序计数器的值保存在了SEPC寄存器；
（3）最后跳转到STVEC寄存器指向的指令。
ecall 指令并没有将 page table 切换为内核页表，也没有切换栈指针，需要进一步执行一些指令才能成功转为内核态。
STVEC寄存器中存储的是 trampoline page 的起始位置。进入内核前，首先需要在该位置处执行一些初始化的操作。例如，切换页表、切换栈指针等操作。需要注意的是，由于用户页表和内核页表都有 trampoline 的索引，且索引的位置是一样的，因此，即使在此刻切换了页表，cpu 也可以正常地找到这个地址，继续在这个位置执行指令。
接下来，cpu 从 trampoline page 处开始进行取指执行。接下来需要保存所有寄存器的值，以便在系统调用后恢复调用前的状态。为此，xv6将进程的所有寄存器的值放到了进程的 trapframe 结构中。
在 kernel/trap.c 中，需要 检查触发trap的原因，以确定相应的处理方式。产生中断的原因有很多，比如系统调用、运算时除以0、使用了一个未被映射的虚拟地址、或者是设备中断等等。这里是因为系统调用，所以以系统调用的方式进行处理。
接下来开始在内核态执行系统调用函数，在 kernel/syscall.c 中取出 a7 寄存器中的函数码，根据该函数码，调用 kernel/sysproc.c 中对应的系统调用函数。
最后，在系统调用函数执行完成后，将保存在 trapframe 中的 SEPC 寄存器的值取出来，从该地址存储的指令处开始执行（保存的值为ecall指令处的PC值加上4，即为 ecall 指令的下一条指令）。随后执行 ret 恢复进入内核态之前的状态，转为用户态。
####Backtrace
1. 该实验相对简单一些，根据提示，先在defs.h中添加声明，然后在kernel/riscv.h 添加读取 s0 寄存器的函数，在 kernel/printf.c 中实现一个 backtrace() 函数。迭代方法，不断循环，输出当前函数的返回地址，直到到达该页表起始地址为止。
####Alarm
1. 做题时感觉有难度，首先尝试与其他题目一样，先在 user/user.h 中添加声明，并在 user/usys.pl 添加 entry，用于生成汇编代码。在 kernel/syscall.h 和kernel/syscall.c 添加函数调用代码。
2. 然后再proc结构体中增加新的变量用于记录时间间隔，经过的时钟数和调用的函数信息，与之相对应的，还需要在alloproc函数中初始化变量，在freeproc中释放内存。
3. 发现test文件中一共有三个测试，先从test0开始看起，发现只需要进入内核，并执行至少一次即可。不需要正确返回也可以通过测试。在sys_sigalarm() 函数中完成对proc的赋值，最后在时钟中断时，添加对应的处理代码，test0就可以通过了。
4. test1和test2的实现需要正确返回到调用前的状态。由于在 ecall 之后的 trampoline 处已经将所有寄存器保存在 trapframe 中，为此，需要添加一个字段，用于保存 trapframe 调用前的所有寄存器的值。这里困扰了我很久，因为在执行好 handler 后，希望的是回到用户调用 handler 前的状态。但那时的状态已经被用来调用 handler 函数了，现在的 trapframe 中存放的是执行 sys_sigreturn 前的 trapframe，如果直接返回到用户态，则找不到之前的状态，无法实现预期。
在 alarmtest 代码中可以看到，每个 handler 函数最后都会调用 sigreturn 函数，用于恢复之前的状态。由于每次使用 ecall 进入中断处理前，都会使用 trapframe 存储当时的寄存器信息，包括时钟中断。因此 trapframe 在每次中断前后都会产生变换，如果要恢复状态，需要额外存储 handler 执行前的 trapframe（即更改返回值为 handler 前的 trapframe），这样，无论中间发生多少次时钟中断或是其他中断，保存的值都不会变。
因此，在 sigreturn 只需要使用存储的状态覆盖调用 sigreturn 时的 trapframe，就可以在 sigreturn 系统调用后恢复到调用 handler 之前的状态。再使用 ret 返回时，就可以返回到执行 handler 之前的用户代码部分。
5. 在实验过程中有一个易错点，如果有一个handler函数正在执行，就不能让第二个handler函数继续执行。为此，再次添加一个字段，用于标记是否有 handler 在执行。

##### 实验结果
通过
![](https://img-blog.csdnimg.cn/043aa905106949e39e2244a8adc908d9.png#pic_center)
