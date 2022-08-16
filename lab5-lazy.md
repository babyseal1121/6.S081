# lab5: lazy page allocation
[源码链接](https://github.com/babyseal1121/6.S081.git)
####实验目的
O/S可以使用页表硬件的许多巧妙技巧之一是延迟分配用户空间堆内存。Xv6应用程序使用sbrk（）系统调用向内核请求堆内存。在我们提供的内核中，sbrk（）分配物理内存并将其映射到进程的虚拟地址空间。内核为一个大请求分配和映射内存可能需要很长时间。例如，假设一个千兆字节由262144 4096字节的页面组成；这是一个巨大的分配数量，即使每个都很便宜。此外，一些程序分配的内存比实际使用的要多（例如，实现稀疏阵列），或者在使用之前分配内存。为了让sbrk（）在这些情况下更快地完成，复杂的内核会延迟分配用户内存。也就是说，sbrk（）不分配物理内存，只是记住分配了哪些用户地址，并在用户页表中将这些地址标记为无效。当进程首次尝试使用延迟分配内存的任何给定页面时，CPU会生成一个页面错误，内核通过分配物理内存、归零和映射来处理该错误。在本实验室中，您将把这个延迟分配特性添加到xv6中。

在开始编码之前，阅读xv6手册的第4章（特别是4.6），以及您可能要修改的相关文件：`kernel/trap.c  kernel/vm.c  kernel/sysproc.c`

要启动实验室，请切换到延迟分支：
```

	$ git fetch
	$ git checkout lazy
	$ make clean
```


####实验内容
#### Eliminate allocation from sbrk() (easy)
你的第一个任务是在sbrk(n) system call实现（函数sys_sbrk()在sysproc.c中）中删除page分配。sbrk(n) system call通过n bytes增长进程的内存尺寸，然后返回最新分配区域的起始位置。你的新sbrk(n)应该只是增加进程尺寸（myproc()->sz）并且返回旧尺寸。它应该不分配内存——所以你该删除对growproc()的调用（但你仍然要增加进程尺寸）。
尝试猜测更改的结果是什么：什么会break？
做此更改，启动xv6，并且在shell输入echo hi。你该看到像下面：

```

	init: starting sh
	$ echo hi
	usertrap(): unexpected scause 0x000000000000000f pid=3
	            sepc=0x0000000000001258 stval=0x0000000000004008
	va=0x0000000000004000 pte=0x0000000000000000
	panic: uvmunmap: not mapped
```
"usertrap():..."信息来自trap.c中的user trap handler；它已经捕获一个异常（不知道如何解决）。确保你理解为什么这个page fault发生。stval=0x0..04008表明导致page fault的虚拟地址是0x4008。



####实验方法
这个实验很简单，就仅仅改动`sys_sbrk()`函数即可，将实际分配内存的函数删除，而仅仅改变进程的`sz`属性

```c
	uint64
	sys_sbrk(void)
	{
	  int addr;
	  int n;
	
	  if(argint(0, &n) < 0)
	    return -1;
	
	  // lazy allocation
	  myproc()->sz += n;
	
	  return addr;
	}
```
### Lazy allocation (moderate)
更改trap.c中的代码对来自用户空间page fault（在faulting address映射一个新分配的物理内存页）做出响应。然后返回到用户空间来让进程继续执行。你该在printf（生成”usertrap():...”信息）之前添加你的代码。修改您需要的任何其他xv6内核代码，以使echo hi正常工作。
以下是一些提示：

	1.	在usertrap()通过看r_scause()是13或15，你可以判断fault是否为page fault。
	2.	r_stval()返回了risc-v stval寄存器的值，包含了导致page fault的虚拟地址
	3.	从vm.c中的uvmalloc()剽窃代码，sbrk()通过growproc()调用uvmalloc。你将需要调用kalloc()和mappages()。
	4.	使用 PGROUNDDOWN(va)让faulting虚拟地址低到页边界
	5.	uvmunmap()将出错；更改它让其在页不映射时，不报错
	6.	如果kernel挂掉，在kernel/kernel.asm中查看sepc
	7.	使用pgtbl lab中的vmprint函数来打印page table内容
	8.	如果你看到错误：”incomplete type proc”，include "spinlock.h" then "proc.h"

如果所有执行正常，你的lazy allocation代码应该导致echo hi正常。你应该至少得到一个page fault（导致懒分配），也可能是两个。



####实验方法：

**(1)**. 修改`usertrap()`(***kernel/trap.c***)函数，使用`r_scause()`判断是否为页面错误，在页面错误处理的过程中，先判断发生错误的虚拟地址（`r_stval()`读取）是否位于栈空间之上，进程大小（虚拟地址从0开始，进程大小表征了进程的最高虚拟地址）之下，然后分配物理内存并添加映射

```c
  uint64 cause = r_scause();
  if(cause == 8) {
    ...
  } else if((which_dev = devintr()) != 0) {
    // ok
  } else if(cause == 13 || cause == 15) {
    // 处理页面错误
    uint64 fault_va = r_stval();  // 产生页面错误的虚拟地址
    char* pa;                     // 分配的物理地址
    if(PGROUNDUP(p->trapframe->sp) - 1 < fault_va && fault_va < p->sz &&
      (pa = kalloc()) != 0) {
        memset(pa, 0, PGSIZE);
        if(mappages(p->pagetable, PGROUNDDOWN(fault_va), PGSIZE, (uint64)pa, PTE_R | PTE_W | PTE_X | PTE_U) != 0) {
          kfree(pa);
          p->killed = 1;
        }
    } else {
      // printf("usertrap(): out of memory!\n");
      p->killed = 1;
    }
  } else {
    ...
  }
```

**(2)**. 修改`uvmunmap()`(***kernel/vm.c***)，之所以修改这部分代码是因为lazy allocation中首先并未实际分配内存，所以当解除映射关系的时候对于这部分内存要略过，而不是使系统崩溃，这部分在课程视频中已经解答。

```c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  ...

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      continue;

    ...
  }
}
```


### Lazytests and Usertests (moderate)
我们已经给你提供了lazytests，一个xv6用户程序，可以测试一些对lazy memory allocator造成压力的特殊场景。更改kernel code让lazytests和usertests都通过测试。

	1.	处理sbrk()负参数
	2.	如果page-faults的虚拟内存地址比sbrk()分配的大，则kill此进程
	3.	正确处理fork() parent-to-child内存拷贝
	4.	处理这么一种情况：进程传递一个来自sbrk()的有效地址给system call例如read or write，但是那些地址的内存尚未分配
	5.	正确处理内存溢出：如果kalloc()在page fault handler中失败，kill当前进程
	6.	处理user stack下的invalid page fault



####实验方法：
**(1)**. 处理`sbrk()`参数为负数的情况，参考之前`sbrk()`调用的`growproc()`程序，如果为负数，就调用`uvmdealloc()`函数，但需要限制缩减后的内存空间不能小于0

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;

  struct proc* p = myproc();
  addr = p->sz;
  uint64 sz = p->sz;

  if(n > 0) {
    // lazy allocation
    p->sz += n;
  } else if(sz + n > 0) {
    sz = uvmdealloc(p->pagetable, sz, sz + n);
    p->sz = sz;
  } else {
    return -1;
  }
  return addr;
}
```

**(2)**. 正确处理`fork`的内存拷贝：`fork`调用了`uvmcopy`进行内存拷贝，所以修改`uvmcopy`如下

```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  ...
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      continue;
    if((*pte & PTE_V) == 0)
      continue;
    ...
  }
  ...
}
```

**(3)**. 还需要继续修改`uvmunmap`，否则会运行出错

```c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  ...

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      continue;
    if((*pte & PTE_V) == 0)
      continue;

    ...
  }
}
```

**(4)**. 处理通过sbrk申请内存后还未实际分配就传给系统调用使用的情况，系统调用的处理会陷入内核，scause寄存器存储的值是8，如果此时传入的地址还未实际分配，就不能走到上文usertrap中判断scause是13或15后进行内存分配的代码，syscall执行就会失败

- 系统调用流程：

  - 陷入内核**==>**`usertrap`中`r_scause()==8`的分支**==>**`syscall()`**==>**回到用户空间

- 页面错误流程：

  - 陷入内核**==>**`usertrap`中`r_scause()==13||r_scause()==15`的分支**==>**分配内存**==>**回到用户空间

因此就需要找到在何时系统调用会使用这些地址，将地址传入系统调用后，会通过`argaddr`函数(***kernel/syscall.c***)从寄存器中读取，因此在这里添加物理内存分配的代码

```c
int
argaddr(int n, uint64 *ip)
{
  *ip = argraw(n);
  struct proc* p = myproc();

  // 处理向系统调用传入lazy allocation地址的情况
  if(walkaddr(p->pagetable, *ip) == 0) {
    if(PGROUNDUP(p->trapframe->sp) - 1 < *ip && *ip < p->sz) {
      char* pa = kalloc();
      if(pa == 0)
        return -1;
      memset(pa, 0, PGSIZE);

      if(mappages(p->pagetable, PGROUNDDOWN(*ip), PGSIZE, (uint64)pa, PTE_R | PTE_W | PTE_X | PTE_U) != 0) {
        kfree(pa);
        return -1;
      }
    } else {
      return -1;
    }
  	}

  	return 0;
	}
```
####实验心得与体会
#####学到的知识点总结
1. 内存懒分配：进程在申请内存时，很难精确地知道所需要的内存多大，因此，进程倾向于申请多于所需要的内存。这样会导致一个问题：有些内存可能一直不会使用，申请了很多内存但是使用的很少。懒分配模式就是解决这样一个问题。解决方法是：分配内存时，只增大进程的内存空间字段值，但并不实际进行内存分配；当该内存段需要使用时，会发现找不到内存页，抛出 page fault 中断，这时再进行物理内存的分配，然后重新执行指令。
![进程地址空间分配图](https://img-blog.csdnimg.cn/fad1ccee0cab4c37bc713fbd8308c332.png#pic_center)
进程创建时，首先为可执行程序分配代码段（text）和数据段（data），然后分配一个无效的页 guard page 用于防止栈溢出。接下来分配进程用户空间栈，xv6 栈的大小是4096，刚好对应一页内存。值得注意的是，栈的生长方向是向下的，sp 是栈指针，初始时指向栈底，即大的地址位置。在栈生长时，栈指针（sp）减小。栈的上面是堆（heap），堆的大小是动态分配的，进程初始化时，堆大小为 0，p->sz 指针指向栈底位置。
#####Eliminate allocation from sbrk()
三个任务的目的都是实现一个功能，递进式实现，有些代码需要在三个任务中不断更改。第一个实验相对比较简单，更改 kernel/sysproc.c 中的 sys_sbrk() 函数，把原来只要申请就分配的逻辑改成申请时仅进行标注，即更改进程的 sz 字段。修改后重新编译xv6，在shell中输入“echo hi”，便会看到因为未分配物理内存而引起的缺页中断。
#####Lazy allocation (moderate)
1. 这个小实验承接上一个，在 kernel/trap.c 中添加代码来处理 page fault 分配物理内存。
2. 在懒分配和eager allocation中有几点不同需要更改，首先，参考 growproc() 函数中调用的 uvmalloc() 代码, 调用 kalloc() 函数为引发 page fault 时记录在寄存器 STVAL 中出错的地址(由 r_stval() 获得)分配一个物理页, 然后调用 mappages() 函数在用户页表中添加虚拟页到物理页的映射. 此处需要使用 PGROUNDDOWN() 来对虚拟地址向下取整。其次，在 xv6 的原本实现中, 由于是 Eager Allocation, 所以不存在分配的虚拟内存不对应物理内存的情况, 因此在 uvmunmap() 函数中取消映射时对于虚拟地址对应的 PTE（页表项） 若是无效的, 便会引发 panic. 而此处改为 Lazy allocation 后, 对于用户进程的虚拟空间, 便会出现有的虚拟内存并不对应实际的物理内存的情况, 即页表中的 PTE 是无效的。因此此时便不能引发 panic, 而是选择跳过该 PTE.
3. 注意usertrap() 中处理 page fault 时要对 va 使用 PGROUNDDOWN() 向下取整，因为其中的终止页 last 是通过 va 计算出来的, 若 va 未向下取整, 则 last==a+1 而非 last==a, 这会导致多映射一页, 从而页面释放时多映射的一页未被取消映射, 从而引发 panic.

#####Lazytests and Usertests (moderate)
1. 实验处理了其中异常情况：处理 sbrk() 负数参数的情况、引发 page fault 的地址低于用户栈和高于分配地址的情况杀死进程、处理 fork() 时的父进程向子进程拷贝内存的情况、处理 read()/write() 使用未分配物理内存的情况、kalloc() 失败时杀死进程
2. 易错点：在处理参数为负的情况时，由于 addr 和 n 是 int 类型, 因此要进行防溢出操作 addr+n>=addr, 避免 addr+n 由正变负造成 p->sz 变小的情况。
3. 将PTE等于0、不存在、无效情况下的panic改为continue跳过，注意要全面修改，不要遗漏。
4. 上述修改导致对于函数的原本逻辑, PTE 无效或者不存在以及无 PTE_U 标志位都会返回 0 表示失败. 但是在 Lazy allocation 的情况下, PTE 无效和不存在是可以被允许的, 因此要对这两种情况进行处理，同时，要考虑到不被允许的情况：但并不是两种情况的所有情形都是允许的, 由于 Lazy allocation 是针对用户堆空间的, 因此需要判断虚拟地址 va 是否在用户堆空间的范围。若满足, 则考虑如何分配内存, 与前面不同的是, read 和 write 为系统调用, 执行到此处时已经在内核模式, 因此这里不会引发 page fault, 因此这里应该和 usertrap() 中的处理类似, 对虚拟地址分配相应的物理页即可。
####实验结果
![](https://img-blog.csdnimg.cn/cc4300e9cfed4be8b4d620e7c6c2d05e.png#pic_center)
![](https://img-blog.csdnimg.cn/6855b8d7d2e2473c8542f3aed94819ea.png#pic_center)
![](https://img-blog.csdnimg.cn/4e5e222f9834472bba820d9eae3d9e90.png#pic_center)
![](https://img-blog.csdnimg.cn/bca9f8f6d8a347398459894f8c45ab95.png#pic_center)

