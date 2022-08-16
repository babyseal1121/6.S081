## lab:page tables
[源码链接](https://github.com/babyseal1121/6.S081.git)
## 前置知识
页表
-. xv6采用三级页表结构


-. 页表缓存
几乎所有的处理器都会对于最近使用过的虚拟地址的翻译结果有缓存。这个缓存被称为：Translation Lookside Buffer（缩写TLB），即页表缓存。当处理器第一次查找一个虚拟地址时，硬件通过3级page table得到最终的PPN，TLB会保存虚拟地址到物理地址的映射关系。这样下一次当你访问同一个虚拟地址时，处理器可以查看TLB，TLB会直接返回物理地址，而不需要通过page table得到结果。

##实验内容

#### Print a page table (easy)
##### 实验要求
在程序刚刚启动(pid==1)时打印输出当前进程的pagetable。
##### 实验步骤
1.在 exec.c 中的return argc之前插入```if(p->pid==1) vmprint(p->pagetable)```，以打印第一个进程的页表

2.在kernel/vm.c中添加vmprint()函数
```

	void vmprinthelper(pagetable_t pagetable, int level)
	{
  	// there are 2^9 = 512 PTEs in a page table.
 	 for(int i = 0; i < 512; i++){
    pte_t pte = pagetable[i];
    if(pte & PTE_V){
      for(int i = 0; i < level; i++)printf(".. ");
      uint64 child = PTE2PA(pte);
      printf("%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
      if (level == 3) 
      	continue;
      else 
      // this PTE points to a lower-level page table.
       vmprinthelper((pagetable_t)child, level + 1);
    		} 
  		}
	}

	void 
	vmprint(pagetable_t pagetable)
	{
  	printf("page table %p\n", pagetable);
  	vmprinthelper(pagetable, 1);
	
```

3.在defs.h中添加
```void            vmprint(pagetable_t);```以便可以从 exec.c 中调用它

#### A kernel page table per process (hard)
##### 实验要求
为每个进程添加一个kernel pagetable来取代之前的global page table
##### 实验步骤
1. 在kernel/proc.h中的struct proc添加一个每个进程的kernel pagetable```pagetable_t kpagetable;```
2. 在vm.c中实现一个三个函数来为每个进程初始化kernel page table,代码如下

```c

	void 
	kvmmapkern(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm)
	{
	  if (mappages(pagetable, va, sz, pa, perm) != 0) 
	    panic("kvmmap");
	}
	
	// proc's version of kvminit
	pagetable_t
	kvmcreate() 
	{
	  pagetable_t pagetable;
	  int i;
	
	  pagetable = uvmcreate();
	  for(i = 1; i < 512; i++) {
	    pagetable[i] = kernel_pagetable[i];
	  }
	
	  // uart registers
	  kvmmapkern(pagetable, UART0, UART0, PGSIZE, PTE_R | PTE_W);
	
	  // virtio mmio disk interface
	  kvmmapkern(pagetable, VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
	
	  // CLINT
	  kvmmapkern(pagetable, CLINT, CLINT, 0x10000, PTE_R | PTE_W);
	
	  // PLIC
	  kvmmapkern(pagetable, PLIC, PLIC, 0x400000, PTE_R | PTE_W);
	
	  return pagetable;
	}
	
	void 
	kvmfree(pagetable_t kpagetale, uint64 sz) 
	{
	  pte_t pte = kpagetale[0];
	  pagetable_t level1 = (pagetable_t) PTE2PA(pte);
	  for (int i = 0; i < 512; i++) {
	    pte_t pte = level1[i];
	    if (pte & PTE_V) {
	      uint64 level2 = PTE2PA(pte);
	      kfree((void *) level2);
	      level1[i] = 0;
	    }
	  }
	  kfree((void *) level1);
	  kfree((void *) kpagetale);
	}
 ```
4. 在desf.h文件中加入上述函数的声明
```c 
	   
	void   kvmmapkern(pagetable_t, uint64, uint64, uint64, int);
	pagetable_t     kvmcreate();
	void          kvmfree(pagetable_t, uint64); 
```

  这是在仿照kvminit，在vm.c中实现函数来为每个进程初始化kernel page table

5.在pro.c文件中，在allocproc()中添加
  ```c
  p->kpagetable = kvmcreate();
  ```

其目的是，调用初始化进程内核页表的函数，让每个procees kernel pagetable拥有这个进程的kernel stack,在allocproc中分配kstack并将其map到p->kpagetable上

6.在freeproc中释放掉process kernel page table
   ```c
   if(p->kpagetable) 
    kvmfree(p->kpagetable, p->sz);
    p->kpagetable = 0;
      }
   ```

7.修改scheduler(),如下
```c
    // ====== solution for pgtbl ---- part 2 ========
        // switch kernel page table
        w_satp(MAKE_SATP(p->kpagetable));
        sfence_vma();
        // ==============================================

        swtch(&c->context, &p->context);

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;

        // ======= solution for pgtbl ---- part 2 ========
        // switch to scheduler's kernel page table
        kvminithart();
        // ===============================================
 ```
 修改scheduler()以将进程的kernel page table加载到satp寄存器中，如果CPU空闲，就使用global kernel page table


#### Simplify copyin/copyinstr (hard)
##### 实验要求
将每个进程的user page table复制到进程kernel page table上，从而让每个进程在copyin的时候不需要再利用process user page table来翻译传入的参数指针，而可以直接利用process kernel page table来dereference
##### 实验步骤
1. 在vm.c中实现一个复制page table的函数来将user page table复制到process kernel page table

```c
	
	void
	kvmmapuser(int pid, pagetable_t kpagetable, pagetable_t upagetable, uint64 newsz, uint64 oldsz)
	{
	  uint64 va;
	  pte_t *upte;
	  pte_t *kpte;
	
	  if(newsz >= PLIC)
	    panic("kvmmapuser: newsz too large");
	
	  for (va = oldsz; va < newsz; va += PGSIZE) {
	    upte = walk(upagetable, va, 0);
	    kpte = walk(kpagetable, va, 1);
	    *kpte = *upte;
	    // because the user mapping in kernel page table is only used for copyin 
	    // so the kernel don't need to have the W,X,U bit turned on
	    *kpte &= ~(PTE_U|PTE_W|PTE_X);
	  }
	}
	// copy PTEs from the user page table into this proc's kernel page table
	void
	kvmmapuser(int pid, pagetable_t kpagetable, pagetable_t upagetable, uint64 newsz, uint64 oldsz)
	{
	  uint64 va;
	  pte_t *upte;
	  pte_t *kpte;
	
	  if(newsz >= PLIC)
	    panic("kvmmapuser: newsz too large");
	
	  for (va = oldsz; va < newsz; va += PGSIZE) {
	    upte = walk(upagetable, va, 0);
	    kpte = walk(kpagetable, va, 1);
	    *kpte = *upte;
	    // because the user mapping in kernel page table is only used for copyin 
	    // so the kernel don't need to have the W,X,U bit turned on
	    *kpte &= ~(PTE_U|PTE_W|PTE_X);
	  }
	}

```

2.在proc.c中的growproc()加入以下，即在每个改变了user page table的地方要调用kvmmapuser使得process kernel page table跟上这个变化。这些地方出现在fork()、exec()和sbrk()需要调用的growproc()。注意要防止user process过大导致virtual address超过PLIC，如下
```c

	   sz = uvmdealloc(p->pagetable, sz, sz + n);
	  }
	  // =========== solution for pgtbl ---- part 3 =============
	  kvmmapuser(p->pid, p->kpagetable, p->pagetable, sz, p->sz);
	  // ========================================================
	  p->sz = sz;
	  return 0;
	}

	   ```
在proc.h中的fork()中添加以下，
```c

	   safestrcpy(np->name, p->name, sizeof(p->name));
	
	  pid = np->pid;
	
	  np->state = RUNNABLE;
	  // ======= solution for pgtbl ---- part 3 =========
	  // remember to map to the new process's kernel page table 
	  kvmmapuser(np->pid, np->kpagetable, np->pagetable, np->sz, 0);
	  // ================================================
	
	  release(&np->lock);
	
	  return pid;
	}
   ```

   在exec.c的exec()return argc 之前，添加以下
   ```c

   	kvmmapuser(p->pid, p->kpagetable,p->pagetable, p->sz, 0);
   ```

3.在proc.h的userinit()中复制process kernel page

```c

	   p->cwd = namei("/");
	
	  p->state = RUNNABLE;
	
	  // =========== solution for pgtbl ---- part 3 =============
	  kvmmapuser(p->pid, p->kpagetable, p->pagetable, p->sz, 0);
	  // ========================================================
	
	  release(&p->lock);
	}
   ```

4.将copyin()和copyinstr()替换为copyin_new()和copyinstr_new()
  ```c

	   int
	copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
	{
	  // ==== solution for pgtbl ---- part 3 =======
	  return copyin_new(pagetable, dst, srcva, len);
	  // ==================================

   ```
   ```c

	   int
	copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
	{
	  // ========= solution for pgtbl ---- part 3 ==========
	  return copyinstr_new(pagetable, dst, srcva, max);
	  // ===================================================
	   ```
5.在defs.h中添加对应声明
```c

	void    kvmmapuser(int, pagetable_t, pagetable_t, uint64, uint64);
	// vmcopyin.c
	int 	copyin_new(pagetable_t, char*, uint64, uint64);
	int     copyinstr_new(pagetable_t, char*, uint64, uint64);

  ```
####实验心得及总结
#### Print a page table 
第一个实验相对来说比较简单，在vm.c中插入一个print函数，然后在defs.h中添加声明，在exec.c文件中加入调用打印页表函数的语句即可。
#### A kernel page table per process
1. 在未修改xv6之前，用户通过系统调用进入内核，这时如果传入用户的指针，会导致因为找不到相应指向的物理地址而失败，因为在虚拟地址到物理地址的转换中，我们使用的是内核kernel pagetable而不是用户自定义的usr pagetable，所以要做的是构造一个usr-kernel-pagetable在用户进入内核时使用，将虚拟地址转换为正确的物理地址。
2. 这里主要是参考kvminit函数，用内核自己pagetable 初始化的方式初始化用户进程的kernel pagetable。在proc结构体中加入内核映射表。
####Simplify copyin/copyinstr
1. 这个任务要求实现对copyin和copyinstr函数的完全替代。通过添加用户地址空间的映射到进程的内核页表, 这样在内核系统调用 copyin() 和 copyinstr() 就不用使用 walk() 函数让操作系统将虚拟地址换为物理地址进行字符拷贝(因为全局内核页表不知道用户地址空间的映射情况); 而是直接通过 MMU 完成虚拟地址到物理地址的转换. 这需要保证用户页表 p->pagetable 变动的同时修改 p->kpagetable, 根据指导书主要修改 fork(), exec(), sbrk() 三个函数.
2. 这里因为两个进程kstack的物理地址不同，所以不能是两个进程的内核页表互相复制，而应该是新进程的内核页表复制自己的用户页表，因为新进程在申请内核页表时内核区已经映射完毕了，所以只需复制用户区即可，这里的复制指浅拷贝，即不是拷贝物理内存而是让两个页表指向同一个物理地址。
3. 指导书中提到, 带有PTE_U标志的PTE不能被内核访问, 因此需要在拷贝用户页表时, 将 PTE 中的 PTE_U 标志清除掉. 这一点比较难以想到.
4. 照葫芦画瓢，直接修改uvmalloc和uvmdealloc函数让其同时处理用户页表和内核页表，不能成功通过测试。原因在于内核页表实际上就是对用户页表的一个引用。在设计操作系统的过程中，架构设计十分重要。
##### 实验结果
![](https://img-blog.csdnimg.cn/6b2a86c76cad44b7ab9a9e1c433d73b8.png#pic_center)