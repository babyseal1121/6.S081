# Lab7: Multithreading
[源码链接](https://github.com/babyseal1121/6.S081.git)s
####实验介绍
**实验目的** 
 
	在本实验中，你需要设计用户级线程切换的上下文
	uthread.c 包含了用户级线程切换的代码
	uthread_switch.S 是你要是实现的保存和恢复线程上下文代码  

**提示**  

	在thread_creat() 和 thread_schedule() 中添加你的代码  
	修改 struct thread 来保存你的上下文   
	你将会在 thread_schedule() 中调用 thread_switch() --该函数应该在 uthread_switch.S 中实现
	thread_switch() 只需要保存和还原唤醒线程和被唤醒线程的上下文  

#### Uthread: switching between threads

####实验内容

本实验是在给定的代码基础上实现用户级线程切换，相比于XV6中实现的内核级线程，这个要简单许多。因为是用户级线程，不需要设计用户栈和内核栈，用户页表和内核页表等等切换，所以本实验中只需要一个类似于`context`的结构，而不需要费尽心机的维护`trapframe`

(1). 定义存储上下文的结构体`tcontext`


	// 用户线程的上下文结构体
	struct tcontext {
	  uint64 ra;
	  uint64 sp;
	
	  // callee-saved
	  uint64 s0;
	  uint64 s1;
	  uint64 s2;
	  uint64 s3;
	  uint64 s4;
	  uint64 s5;
	  uint64 s6;
	  uint64 s7;
	  uint64 s8;
	  uint64 s9;
	  uint64 s10;
	  uint64 s11;
	};


(2). 修改`thread`结构体，添加`context`字段


	struct thread {
	  char            stack[STACK_SIZE];  /* the thread's stack */
	  int             state;              /* FREE, RUNNING, RUNNABLE */
	  struct tcontext context;            /* 用户进程上下文 */
	};


(3). 模仿***kernel/swtch.S，***在***kernel/uthread_switch.S***中写入如下代码


	/*
	* save the old thread's registers,
	* restore the new thread's registers.
	*/
	
	.globl thread_switch
	thread_switch:
	    /* YOUR CODE HERE */
	    sd ra, 0(a0)
	    sd sp, 8(a0)
	    sd s0, 16(a0)
	    sd s1, 24(a0)
	    sd s2, 32(a0)
	    sd s3, 40(a0)
	    sd s4, 48(a0)
	    sd s5, 56(a0)
	    sd s6, 64(a0)
	    sd s7, 72(a0)
	    sd s8, 80(a0)
	    sd s9, 88(a0)
	    sd s10, 96(a0)
	    sd s11, 104(a0)
	
	    ld ra, 0(a1)
	    ld sp, 8(a1)
	    ld s0, 16(a1)
	    ld s1, 24(a1)
	    ld s2, 32(a1)
	    ld s3, 40(a1)
	    ld s4, 48(a1)
	    ld s5, 56(a1)
	    ld s6, 64(a1)
	    ld s7, 72(a1)
	    ld s8, 80(a1)
	    ld s9, 88(a1)
	    ld s10, 96(a1)
	    ld s11, 104(a1)
	    ret    /* return to ra */


(4). 修改`thread_scheduler`，添加线程切换语句



	...
	if (current_thread != next_thread) {         /* switch threads?  */
	  ...
	  /* YOUR CODE HERE */
	  thread_switch((uint64)&t->context, (uint64)&current_thread->context);
	} else
	  next_thread = 0;


(5). 在`thread_create`中对`thread`结构体做一些初始化设定，主要是`ra`返回地址和`sp`栈指针.



	// YOUR CODE HERE
	t->context.ra = (uint64)func;                   // 设定函数返回地址
	t->context.sp = (uint64)t->stack + STACK_SIZE;  // 设定栈指针

Q: thread_switch needs to save/restore only the callee-save registers. Why?   

A: 这里仅需保存被调用者保存(callee-save)寄存器的原因和 xv6 中内核线程切换时仅保留 callee-save 寄存器的原因是相同的. 由于 thread_switch() 一定由其所在的 C 语言函数调用, 因此函数的调用规则是满足 xv6 的函数调用规则的, 对于其它 caller-save 寄存器都会被保存在线程的堆栈上, 在切换后的线程上下文恢复时可以直接从切换后线程的堆栈上恢复 caller-save 寄存器的值. 由于 callee-save 寄存器是由被调用函数即 thread_switch() 进行保存的, 在函数返回时已经丢失, 因此需要额外保存这些寄存器的内容.

#### Using threads
####实验方法
####实验内容

来看一下程序的运行过程：设定了五个散列桶，根据键除以5的余数决定插入到哪一个散列桶中，插入方法是头插法，下面是图示

不支持在 Docs 外粘贴 block

这个实验比较简单，首先是问为什么为造成数据丢失：

> 假设现在有两个线程T1和T2，两个线程都走到put函数，且假设两个线程中key%NBUCKET相等，即要插入同一个散列桶中。两个线程同时调用insert(key, value, &table[i], table[i])，insert是通过头插法实现的。如果先insert的线程还未返回另一个线程就开始insert，那么前面的数据会被覆盖

因此只需要对插入操作上锁即可

(1). 为每个散列桶定义一个锁，将五个锁放在一个数组中，并进行初始化


	pthread_mutex_t lock[NBUCKET] = { PTHREAD_MUTEX_INITIALIZER }; // 每个散列桶一把锁


(2). 在`put`函数中对`insert`上锁



	if(e){
	    // update the existing key.
	    e->value = value;
	} else {
	    pthread_mutex_lock(&lock[i]);
	    // the new is new.
	    insert(key, value, &table[i], table[i]);
	    pthread_mutex_unlock(&lock[i]);
	}


#### Barrier
####实验方法
####实验内容

保证下一个round的操作不会影响到上一个还未结束的round中的数据



	static void 
	barrier()
	{
	  // 申请持有锁
	  pthread_mutex_lock(&bstate.barrier_mutex);
	
	  bstate.nthread++;
	  if(bstate.nthread == nthread) {
	    // 所有线程已到达
	    bstate.round++;
	    bstate.nthread = 0;
	    pthread_cond_broadcast(&bstate.barrier_cond);
	  } else {
	    // 等待其他线程
	    // 调用pthread_cond_wait时，mutex必须已经持有
	    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
	  }
	  // 释放锁
	  pthread_mutex_unlock(&bstate.barrier_mutex);
	}

####实验心得体会及总结
####学到的知识总结
**线程调度**  

(1). 首先是用户线程接收到了时钟中断，强迫CPU从用户空间进程切换到内核，同时在 trampoline 代码中，保存当前寄存器状态到 trapframe 中；  
(2). 在 usertrap 处理中断时，切换到了该进程对应的内核线程；  
(3)内核线程在内核中，先做一些操作，然后调用 swtch 函数，保存用户进程对应的内核线程的寄存器至 context 对象；  
(4)swtch 函数并不是直接从一个内核线程切换到另一个内核线程；而是先切换到当前 cpu 对应的调度器线程，之后就在调度器线程的 context 下执行 schedulder 函数中；   
(5)schedulder 函数会再次调用 swtch 函数，切换到下一个内核线程中，由于该内核线程肯定也调用了 swtch 函数，所以之前的 swtch 函数会被恢复，并返回到内核线程所对应进程的系统调用或者中断处理程序中。  
(6)当内核程序执行完成之后，trapframe 中的用户寄存器会被恢复，完成线程调度。
线程调度过程如下图所示：
![](https://pic3.zhimg.com/80/v2-8755dadfc6f65aef397bc7efa483393e_720w.jpg)
####switching between threads
1. 上下文切换：首先是 uthread_switch.S 中实现上下文切换。先将一些寄存器保存到第一个参数中，然后再将第二个参数中的寄存器加载进来。
2. 添加代码到 thread_create() 函数时，传递的 thread_create() 参数 func 需要记录, 这样在线程运行时才能运行该函数, 此外线程的栈结构是独立的, 在运行函数时要在线程自己的栈上, 因此也要初始化线程的栈指针. 而在线程进行调度切换时, 同样需要保存和恢复寄存器状态, 而上述二者实际上分别对应着 ra 和 sp 寄存器, 在线程初始化进行设置, 这样在后续调度切换时便能保持其正确性。
3. 
####Using threads
注意：在put时候添加锁防止条件竞争
####Barrier
此处主要涉及互斥锁和条件变量配合达到线程同步.
首先条件变量的操作需要在互斥锁锁定的临界区内.
然后进行条件判断, 此处即判断是否所有的线程都进入了 barrier() 函数, 若不满足则使用 pthread_cond_wait() 将当前线程休眠, 等待唤醒; 若全部线程都已进入 barrier() 函数, 则最后进入的线程会调用 pthread_cond_broadcast() 唤醒其他由条件变量休眠的线程继续运行.
需要注意的是, 对于 pthread_cond_wait() 涉及三个操作: 原子的释放拥有的锁并阻塞当前线程, 这两个操作是原子的; 第三个操作是由条件变量唤醒后会再次获取锁.
####感想
在这次实验中实现了用户级线程调度，对于线程调度机制有了更深刻的了解，学习了一边线程的切换过程。
####实验结果
![](https://img-blog.csdnimg.cn/855fcab60d00466ca358a22ff279cb7e.png#pic_center)