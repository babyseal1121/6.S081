# lab6: Copy-on-Write Fork for xv6
[源码链接](https://github.com/babyseal1121/6.S081.git)
####实验介绍
虚拟内存提供一定程度的重定向：kernel可以通过标记PTEs无效、只读（导致page faults）来中断内存引用；kernel也可以通过改变地址含义（通过更改PTES）。在电脑系统中有个说法：系统问题可以通过一定程度的重定向解决。lazy allocation lab提供了一个例子。这个lab探索另外的例子：copy-on write fork。

**The problem**
xv6中的fork()系统调用，复制parent进程所有的用户空间内存到child。如果parent是非常大的，copying将花费很长时间。糟糕的是，这个工作通常是大量浪费的；例如，fork()后紧接着是exec()在child进程中，这将导致child会丢弃拷贝的内存，可能绝大多数都不使用。另一方面，如果parent和child使用一个page，并且其中一个或两个写，那么确实需要一个副本。

**The solution**
copy-on-write（COW）fork()的目的是：推迟对child的分配和拷贝物理内存页，直到拷贝确实需要。
COW fork()仅仅给child创建一个pagetable，其用户内存的PTEs指向parent的物理页。COW fork()标记parent和child的所有用户内存PTEs是不可写的。当某个进程尝试写其中一个COW页时，cpu将强制一个page fault。kernel page-fault handler检测这种情形，为faulting进程分配一页物理内存，复制原始页到新页，更改faulting进程相关PTE来指向新页，这次让PTE标记为可写。当page fault handler返回时，用户进程将能够向拷贝页写入。
COW fork()让物理页（实现用户内存）的释放更有技巧性。一个给定物理页可能被多个进程的page table指向，仅应该在最后的指向消失时，才释放物理页。

####实验内容
#### Implement copy-on write(hard)
你的任务是在xv6 kernel实现copy-on fork。如果更改的kernel code通过cowtests和usertests，则成功了。
为了帮你测试你的实现，我们已经提供了一个xv6程序（cowtest，源码在user/cowtest.c）。cowtest运行多个tests，未改xv6时第一个就会失败。

	1.	更改uvmcopy()来映射parent物理页到child，而不是分配新页。清除parent和child PTEs的PTE_W。
	2.	更改usertrap()来识别page faults。当一个page-fault发生在一个COW page，通过kalloc()分配一个新页，复制旧页到新页，并且安装新页到PTE（设置PTE_W）。
	3.	确保每个物理页被释放，当最后的PTE指向移除时。这么做的一个好方式是：对每个物理页保存一个“reference count”，表明指向此物理页的page tables数量。设置page的reference数目为1，当kalloc()分配它时。当fork导致child分享此页时,增加page的reference数目；每次任意进程从page table中删除此page时,减少page的reference数目。如果它的reference数目为0,kfree()应该仅仅放置一个page在free list最后。将这些计数放到一个固定长度的整数数组中。你将不得不发明一个模型：如何索引数组，如何选择它的尺寸。例如：你可以用页物理地址除以4096对数组进行索引，并给数组一些元素，这些元素，通过kalloc.c中的kinit()放在free list中的页
	4.	当遇到一个COW page时，更改copyout()，使用与page fault相同的方法。

提示:

	1.	lazy page allocation lab可能已经让你熟悉了一些xv6 kernel代码（与copy-on-write相关的）。然而，你不应该让本实验基于lazy allocation的方案。而是根据上面引导的，开始一个新的xv6拷贝。
	2.	使用RISC-V PTE的预留标志位(RSW (reserved for software) bits)，来记录每个PTE是不是一个COW映射，这可能是有用的。
	3.	usertests探索一些cowtest没有测试到的地方，不要忘记核对两个测试都通过。
	4.	一些对页表标志位有帮助的宏指令和定义在kernel/riscv.h下面。
	5.	如果一个COW page fault发生，但没有空闲内存，此进程应该被杀掉。
	

####实验方法
(1). 在kernel/riscv.h中选取PTE中的保留位定义标记一个页面是否为COW Fork页面的标志位
```c 

	// 记录应用了COW策略后fork的页面
	define PTE_F (1L << 8)
```

(2). 在kalloc.c中进行如下修改
定义引用计数的全局变量ref，其中包含了一个自旋锁和一个引用计数数组，由于ref是全局变量，会被自动初始化为全0。
这里使用自旋锁是考虑到这种情况：进程P1和P2共用内存M，M引用计数为2，此时CPU1要执行fork产生P1的子进程，CPU2要终止P2，那么假设两个CPU同时读取引用计数为2，执行完成后CPU1中保存的引用计数为3，CPU2保存的计数为1，那么后赋值的语句会覆盖掉先赋值的语句，从而产生错误
c struct ref_stru { struct spinlock lock; int cnt[PHYSTOP / PGSIZE]; // 引用计数 } ref;
在kinit中初始化ref的自旋锁
c void kinit() { initlock(&kmem.lock, "kmem"); initlock(&ref.lock, "ref"); freerange(end, (void*)PHYSTOP); }
修改kalloc和kfree函数，在kalloc中初始化内存引用计数为1，在kfree函数中对内存引用计数减1，如果引用计数为0时才真正删除
```c 
	void kfree(void *pa) { struct run *r;
	if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP) panic("kfree");
	// 只有当引用计数为0了才回收空间 // 否则只是将引用计数减1 acquire(&ref.lock); if(--ref.cnt[(uint64)pa / PGSIZE] == 0) { release(&ref.lock);
	r = (struct run*)pa;
	
	// Fill with junk to catch dangling refs.
	memset(pa, 1, PGSIZE);
	
	acquire(&kmem.lock);
	r->next = kmem.freelist;
	kmem.freelist = r;
	release(&kmem.lock);
	} else { release(&ref.lock); } } void * kalloc(void) { struct run *r;
	acquire(&kmem.lock); r = kmem.freelist; if(r) { kmem.freelist = r->next; acquire(&ref.lock); ref.cnt[(uint64)r / PGSIZE] = 1; // 将引用计数初始化为1 release(&ref.lock); } release(&kmem.lock);
	if(r) memset((char)r, 5, PGSIZE); // fill with junk return (void)r; } 
```

添加如下四个函数，详细说明已在注释中，这些函数中用到了walk，记得在defs.h中添加声明，最后也需要将这些函数的声明添加到defs.h，在cowalloc中，读取内存引用计数，如果为1，说明只有当前进程引用了该物理内存（其他进程此前已经被分配到了其他物理页面），就只需要改变PTE使能PTE_W；否则就分配物理页面，并将原来的内存引用计数减1。该函数需要返回物理地址，这将在copyout中使用到。
```c 

	/** * @brief cowpage 判断一个页面是否为COW页面 * @param pagetable 指定查询的页表 * @param va 虚拟地址 * @return 0 是 -1 不是 / int cowpage(pagetablet pagetable, uint64 va) { if(va >= MAXVA) return -1; ptet pte = walk(pagetable, va, 0); if(pte == 0) return -1; if((pte & PTEV) == 0) return -1; return (pte & PTEF ? 0 : -1); }
	/** * @brief cowalloc copy-on-write分配器 * @param pagetable 指定页表 * @param va 指定的虚拟地址,必须页面对齐 * @return 分配后va对应的物理地址，如果返回0则分配失败 / void cowalloc(pagetable_t pagetable, uint64 va) { if(va % PGSIZE != 0) return 0;
	uint64 pa = walkaddr(pagetable, va); // 获取对应的物理地址 if(pa == 0) return 0;
	pte_t* pte = walk(pagetable, va, 0); // 获取对应的PTE
	if(krefcnt((char*)pa) == 1) { // 只剩一个进程对此物理地址存在引用 // 则直接修改对应的PTE即可 pte |= PTEW; pte &= ~PTEF; return (void)pa; } else { // 多个进程对物理内存存在引用 // 需要分配新的页面，并拷贝旧页面的内容 char mem = kalloc(); if(mem == 0) return 0;
	// 复制旧页面内容到新页
	memmove(mem, (char*)pa, PGSIZE);
	
	// 清除PTE_V，否则在mappagges中会判定为remap
	*pte &= ~PTE_V;
	
	// 为新页面添加映射
	if(mappages(pagetable, va, PGSIZE, (uint64)mem, (PTE_FLAGS(*pte) | PTE_W) & ~PTE_F) != 0) {
	  kfree(mem);
	  *pte |= PTE_V;
	  return 0;
	}
	
	// 将原来的物理内存引用计数减1
	kfree((char*)PGROUNDDOWN(pa));
	return mem;
	} }
	/** * @brief krefcnt 获取内存的引用计数 * @param pa 指定的内存地址 * @return 引用计数 / int krefcnt(void pa) { return ref.cnt[(uint64)pa / PGSIZE]; }
	/** * @brief kaddrefcnt 增加内存的引用计数 * @param pa 指定的内存地址 * @return 0:成功 -1:失败 / int kaddrefcnt(void pa) { if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP) return -1; acquire(&ref.lock); ++ref.cnt[(uint64)pa / PGSIZE]; release(&ref.lock); return 0; }
```

修改freerange
```c

	void freerange(void *pa_start, void *pa_end) 
	{ 
		char *p; 
		p = (char*)PGROUNDUP((uint64)pa_start); 
		for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE) 
		{ // 在kfree中将会对cnt[]减1，这里要先设为1，否则就会减成负数 
			ref.cnt[(uint64)p / PGSIZE] = 1; 
			kfree(p); 
		} 
	}

```

(3). 修改uvmcopy，不为子进程分配内存，而是使父子进程共享内存，但禁用PTE_W，同时标记PTE_F，记得调用kaddrefcnt增加引用计数
```c 

	int uvmcopy(pagetablet old, pagetablet new, uint64 sz) { pte_t *pte; uint64 pa, i; uint flags;
	for(i = 0; i < sz; i += PGSIZE){ if((pte = walk(old, i, 0)) == 0) panic("uvmcopy: pte should exist"); if((pte & PTEV) == 0) panic("uvmcopy: page not present"); pa = PTE2PA(pte); flags = PTEFLAGS(*pte);
	// 仅对可写页面设置COW标记
	if(flags & PTE_W) {
	  // 禁用写并设置COW Fork标记
	  flags = (flags | PTE_F) & ~PTE_W;
	  *pte = PA2PTE(pa) | flags;
	}
	
	if(mappages(new, i, PGSIZE, pa, flags) != 0) {
	  uvmunmap(new, 0, i / PGSIZE, 1);
	  return -1;
	}
	// 增加内存的引用计数
	kaddrefcnt((char*)pa);
	} return 0; } 
```

(4). 修改usertrap，处理页面错误
```c

	uint64 cause = r_scause(); 
	if(cause == 8) 
		{ ... } 
	else if((which_dev = devintr()) != 0)
		{ // ok } 
	else if(cause == 13 || cause == 15) 
		{ uint64 fault_va = r_stval(); // 获取出错的虚拟地址 
	if(fault_va >= p->sz || cowpage(p->pagetable, fault_va) != 0 || cowalloc(p->pagetable, PGROUNDDOWN(fault_va)) == 0) 
		p->killed = 1; } 
	else 
		{ ... }

```


(5). 在copyout中处理相同的情况，如果是COW页面，需要更换pa0指向的物理地址
```c 

	while(len > 0){ va0 = PGROUNDDOWN(dstva); pa0 = walkaddr(pagetable, va0);
	// 处理COW页面的情况 
	if(cowpage(pagetable, va0) == 0) 
	{ // 更换目标物理地址 pa0 = (uint64)cowalloc(pagetable, va0); }
	if(pa0 == 0) return -1;
	... } 
```
###实验心得体会及总结
####学到的知识点总结
1. copy-on-write 是指当你创建子进程时，并不实际复制父进程的空间地址的内容到新的物理内存，而是将页表项改为只读，并在第一次父/子进程要改变页面内容时才进行复制，并将原来的页面解锁为可读写。这种做法节省了大量的物理内存，尤其是现在大部分程序在 fork() 和通常立马会接上 exce() 载入新的程序 。写时复制 (COW) fork() 的目标是推迟为子进程分配和复制物理内存页面，直到实际需要副本（如果有的话）。
2. COW fork() 只为子级创建一个页表，用户内存的 PTE 指向父级的物理页面。 COW fork() 将 parent 和 child 中的所有用户 PTE 标记为不可写。当任一进程尝试写入这些 COW 页之一时，CPU 将强制发生页错误。内核页面错误处理程序检测到这种情况，为出错进程分配物理内存页面，将原始页面复制到新页面中，并修改出错进程中的相关 PTE 以引用新页面，这次使用PTE 标记为可写。当页面错误处理程序返回时，用户进程将能够写入它的页面副本。
3. 当发生page fault时，引发page fault的地址存储在寄存器stval中，引发pagefault的原因存储在寄存器scause中，引发page fault的指令地址在sepc中。

####cow
1. 总体来说本实验的思路跟lazy alloc比较类似，最重要的思想就是子进程一开始没有自己的物理页，原来的被分享的物理页是只读的，一旦出现页错误就要为这个子进程分配新的物理空间，并且将它的虚拟地址映射过去，然后将这些物理页标记成为可写的。
2. 难点：要设置的标志一会是考虑reference count一会又要考虑PTE_COW，在分配和释放的时候总是想不清楚。在释放PTE时有可能出现如果refcount已经是0，说明系统出现了多次释放同一块内存的bug。减完之后再检测一下refcount是不是0，如果是0则释放即可。
3. 难点：锁的添加。不用锁的话会内存泄漏，应该是多进程竞争扰乱了引用计数的加减。
4. 易错点：需要处理的细节很多，经常在莫名其妙的地方发生错误。在把子进程的虚拟地址映射到父进程的物理页表上时，获取父进程每页对应的PTE表项，清除掉父进程的PTE_W位，提取出它的flag，在给子进程的虚拟地址建立页表映射的时候使用这个flag，这样子进程的各个PTE和父进程就相同了。
4. 易错点：如果传入了非法地址，及时退出；kalloc()不能保证一定能分配出一块内存，所以要记得检测pa2是不是0，如果分配不出来内存要退出。
5. 感想：这个实验要修改的细节很多，尤其是对于一些错误的处理，要多测试几次，才能面面俱到。
####实验结果
![](https://img-blog.csdnimg.cn/ca25ff7b027c40ddbb9e369925850dd9.png#pic_center)