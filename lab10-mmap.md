# Lab10: mmap
[源码链接](https://github.com/babyseal1121/6.S081.git)
####实验介绍

**实验目的** 
 
mmap和munmap系统调用允许UNIX程序对它们的地址空间进行详细的控制。它们可以用于在进程之间共享内存，将文件映射到进程地址空间，以及作为用户级页面错误方案的一部分，比如在讲座中讨论的垃圾收集算法。在本实验中，您将向xv6添加mmap和munmap，重点关注内存映射文件。    

**提示**  

1. 首先将_mmaptest添加到UPROGS，并调用mmap和munmap系统，以便编译user/mmaptest.c。现在，只返回来自mmap和munmap的错误。我们在kernel/fcntl.h中为你定义了PROT_READ等。运行mmaptest，它将在第一次调用mmap时失败。

2. 延迟填充页表，以响应页错误。也就是说，mmap不应该分配物理内存或读取文件。相反，应该在usertrap中的(或由usertrap调用的)页面错误处理代码中这样做，就像在lazy page allocation实验室中那样。使用lazy的原因是为了确保大文件的mmap速度快，并且mmap可以用于大于物理内存的文件。
	
3. 跟踪mmap为每个进程映射了什么。定义一个对应于VMA(虚拟内存区域)的结构，在Lecture 15中描述，记录地址，长度，权限，文件等由mmap创建的虚拟内存范围。因为xv6内核在内核中没有内存分配器，所以可以声明一个固定大小的vma数组，并根据需要从该数组进行分配。大小为16就足够了。
	
4. 实现mmap:在进程的地址空间中找到一个未使用的区域来映射文件，并在映射区域的进程表中添加一个VMA。VMA应该包含一个指向被映射文件的结构文件的指针;mmap应该增加文件的引用计数，这样当文件关闭时结构不会消失(提示:参见filedup)。运行maptest:第一个mmap应该成功，但是第一次访问被映射的内存将导致页面错误并杀死mmaptest。
	
5. 添加代码在映射区域中导致页错误，以便分配物理内存页，将相关文件的4096字节读入该页，并将其映射到用户地址空间。使用readi读取文件，它接受一个偏移量参数，在该参数处读取文件(但您必须锁定/解锁传递给readi的inode)。不要忘记正确设置页面上的权限。mmaptest运行;它应该会到达第一个munmap。
	
6. 实现munmap:查找地址范围的VMA，并解除指定页面的映射(提示:使用uvmunmap)。如果munmap删除了前一个mmap的所有页面，它应该减少相应结构文件的引用计数。如果修改了未映射的页面，并且将文件映射为MAP_SHARED，则将该页面写回该文件。从filewrite中寻找灵感。
	
7. 理想情况下，您的实现只写回程序实际修改的MAP_SHARED页。RISC-V PTE中的脏位(D)表示是否写入了一个页面。然而，mmaptest并不检查非脏页是否被写回;因此你可以不用看D位就可以把页面写回去。
	
8. 修改exit以取消进程映射区域的映射，就像调用了munmap一样。mmaptest运行;Mmap_test应该通过，但可能无法通过fork_test。
	
9. 修改fork以确保子节点具有与父节点相同的映射区域。不要忘记增加VMA的结构文件的引用计数。在子节点的页错误处理程序中，可以分配一个新的物理页，而不是与父节点共享一个页。后者更酷一些，但需要更多的实现工作。mmaptest运行;它应该同时通过mmap_test和fork_test。

####实验步骤
(1). 根据提示1，首先是配置`mmap`和`munmap`系统调用，此前已进行过多次类似流程，不再赘述。在***kernel/fcntl.h***中定义了宏，只有在定义了`LAB_MMAP`时这些宏才生效，而`LAB_MMAP`是在编译时在命令行通过gcc的`-D`参数定义的


	void* mmap(void* addr, int length, int prot, int flags, int fd, int offset);
	int munmap(void* addr, int length);


(2). 根据提示3，定义VMA结构体，并添加到进程结构体中


	#define NVMA 16
	// 虚拟内存区域结构体
	struct vm_area {
	  int used;           // 是否已被使用
	  uint64 addr;        // 起始地址
	  int len;            // 长度
	  int prot;           // 权限
	  int flags;          // 标志位
	  int vfd;            // 对应的文件描述符
	  struct file* vfile; // 对应文件
	  int offset;         // 文件偏移，本实验中一直为0
	};
	
	struct proc {
	  ...
	  struct vm_area vma[NVMA];    // 虚拟内存区域
	}


(3). 在allocproc中将vma数组初始化为全0


	static struct proc*
	allocproc(void)
	{
	  ...
	
	found:
	  ...
	
	  memset(&p->vma, 0, sizeof(p->vma));
	  return p;
	}
	

(4). 根据提示2、3、4，参考lazy实验中的分配方法（将当前`p->sz`作为分配的虚拟起始地址，但不实际分配物理页面），此函数写在***sysfile.c***中就可以使用静态函数`argfd`同时解析文件描述符和`struct file`


	uint64
	sys_mmap(void) {
	  uint64 addr;
	  int length;
	  int prot;
	  int flags;
	  int vfd;
	  struct file* vfile;
	  int offset;
	  uint64 err = 0xffffffffffffffff;
	
	  // 获取系统调用参数
	  if(argaddr(0, &addr) < 0 || argint(1, &length) < 0 || argint(2, &prot) < 0 ||
	    argint(3, &flags) < 0 || argfd(4, &vfd, &vfile) < 0 || argint(5, &offset) < 0)
	    return err;
	
	  // 实验提示中假定addr和offset为0，简化程序可能发生的情况
	  if(addr != 0 || offset != 0 || length < 0)
	    return err;
	
	  // 文件不可写则不允许拥有PROT_WRITE权限时映射为MAP_SHARED
	  if(vfile->writable == 0 && (prot & PROT_WRITE) != 0 && flags == MAP_SHARED)
	    return err;
	
	  struct proc* p = myproc();
	  // 没有足够的虚拟地址空间
	  if(p->sz + length > MAXVA)
	    return err;
	
	  // 遍历查找未使用的VMA结构体
	  for(int i = 0; i < NVMA; ++i) {
	    if(p->vma[i].used == 0) {
	      p->vma[i].used = 1;
	      p->vma[i].addr = p->sz;
	      p->vma[i].len = length;
	      p->vma[i].flags = flags;
	      p->vma[i].prot = prot;
	      p->vma[i].vfile = vfile;
	      p->vma[i].vfd = vfd;
	      p->vma[i].offset = offset;
	
	      // 增加文件的引用计数
	      filedup(vfile);
	
	      p->sz += length;
	      return p->vma[i].addr;
	    }
	  }
	
	  return err;
	}


(5). 根据提示5，此时访问对应的页面就会产生页面错误，需要在`usertrap`中进行处理，主要完成三项工作：分配物理页面，读取文件内容，添加映射关系


	void
	usertrap(void)
	{
	  ...
	  if(cause == 8) {
	    ...
	  } else if((which_dev = devintr()) != 0){
	    // ok
	  } else if(cause == 13 || cause == 15) {
	#ifdef LAB_MMAP
	    // 读取产生页面故障的虚拟地址，并判断是否位于有效区间
	    uint64 fault_va = r_stval();
	    if(PGROUNDUP(p->trapframe->sp) - 1 < fault_va && fault_va < p->sz) {
	      if(mmap_handler(r_stval(), cause) != 0) p->killed = 1;
	    } else
	      p->killed = 1;
	#endif
	  } else {
	    ...
	  }
	
	  ...
	}
	
	/**
	 * @brief mmap_handler 处理mmap惰性分配导致的页面错误
	 * @param va 页面故障虚拟地址
	 * @param cause 页面故障原因
	 * @return 0成功，-1失败
	 */
	int mmap_handler(int va, int cause) {
	  int i;
	  struct proc* p = myproc();
	  // 根据地址查找属于哪一个VMA
	  for(i = 0; i < NVMA; ++i) {
	    if(p->vma[i].used && p->vma[i].addr <= va && va <= p->vma[i].addr + p->vma[i].len - 1) {
	      break;
	    }
	  }
	  if(i == NVMA)
	    return -1;
	
	  int pte_flags = PTE_U;
	  if(p->vma[i].prot & PROT_READ) pte_flags |= PTE_R;
	  if(p->vma[i].prot & PROT_WRITE) pte_flags |= PTE_W;
	  if(p->vma[i].prot & PROT_EXEC) pte_flags |= PTE_X;
	
	
	  struct file* vf = p->vma[i].vfile;
	  // 读导致的页面错误
	  if(cause == 13 && vf->readable == 0) return -1;
	  // 写导致的页面错误
	  if(cause == 15 && vf->writable == 0) return -1;
	
	  void* pa = kalloc();
	  if(pa == 0)
	    return -1;
	  memset(pa, 0, PGSIZE);
	
	  // 读取文件内容
	  ilock(vf->ip);
	  // 计算当前页面读取文件的偏移量，实验中p->vma[i].offset总是0
	  // 要按顺序读读取，例如内存页面A,B和文件块a,b
	  // 则A读取a，B读取b，而不能A读取b，B读取a
	  int offset = p->vma[i].offset + PGROUNDDOWN(va - p->vma[i].addr);
	  int readbytes = readi(vf->ip, 0, (uint64)pa, offset, PGSIZE);
	  // 什么都没有读到
	  if(readbytes == 0) {
	    iunlock(vf->ip);
	    kfree(pa);
	    return -1;
	  }
	  iunlock(vf->ip);
	
	  // 添加页面映射
	  if(mappages(p->pagetable, PGROUNDDOWN(va), PGSIZE, (uint64)pa, pte_flags) != 0) {
	    kfree(pa);
	    return -1;
	  }
	
	  return 0;
	}


(6). 根据提示6实现`munmap`，且提示7中说明无需查看脏位就可写回


	uint64
	sys_munmap(void) {
	  uint64 addr;
	  int length;
	  if(argaddr(0, &addr) < 0 || argint(1, &length) < 0)
	    return -1;
	
	  int i;
	  struct proc* p = myproc();
	  for(i = 0; i < NVMA; ++i) {
	    if(p->vma[i].used && p->vma[i].len >= length) {
	      // 根据提示，munmap的地址范围只能是
	      // 1. 起始位置
	      if(p->vma[i].addr == addr) {
	        p->vma[i].addr += length;
	        p->vma[i].len -= length;
	        break;
	      }
	      // 2. 结束位置
	      if(addr + length == p->vma[i].addr + p->vma[i].len) {
	        p->vma[i].len -= length;
	        break;
	      }
	    }
	  }
	  if(i == NVMA)
	    return -1;
	
	  // 将MAP_SHARED页面写回文件系统
	  if(p->vma[i].flags == MAP_SHARED && (p->vma[i].prot & PROT_WRITE) != 0) {
	    filewrite(p->vma[i].vfile, addr, length);
	  }
	
	  // 判断此页面是否存在映射
	  uvmunmap(p->pagetable, addr, length / PGSIZE, 1);
	
	
	  // 当前VMA中全部映射都被取消
	  if(p->vma[i].len == 0) {
	    fileclose(p->vma[i].vfile);
	    p->vma[i].used = 0;
	  }
	
	  return 0;
	}


(7). 回忆lazy实验中，如果对惰性分配的页面调用了`uvmunmap`，或者子进程在fork中调用`uvmcopy`复制了父进程惰性分配的页面都会导致panic，因此需要修改`uvmunmap`和`uvmcopy`检查`PTE_V`后不再`panic`


	if((*pte & PTE_V) == 0)
	  continue;


(8). 根据提示8修改`exit`，将进程的已映射区域取消映射


	void
	exit(int status)
	{
	  // Close all open files.
	  for(int fd = 0; fd < NOFILE; fd++){
	    ...
	  }
	
	  // 将进程的已映射区域取消映射
	  for(int i = 0; i < NVMA; ++i) {
	    if(p->vma[i].used) {
	      if(p->vma[i].flags == MAP_SHARED && (p->vma[i].prot & PROT_WRITE) != 0) {
	        filewrite(p->vma[i].vfile, p->vma[i].addr, p->vma[i].len);
	      }
	      fileclose(p->vma[i].vfile);
	      uvmunmap(p->pagetable, p->vma[i].addr, p->vma[i].len / PGSIZE, 1);
	      p->vma[i].used = 0;
	    }
	  }
	
	  begin_op();
	  iput(p->cwd);
	  end_op();
	  ...
	}


(9). 根据提示9，修改`fork`，复制父进程的VMA并增加文件引用计数


	int
	fork(void)
	{
	 // increment reference counts on open file descriptors.
	  for(i = 0; i < NOFILE; i++)
	    ...
	  ...
	
	  // 复制父进程的VMA
	  for(i = 0; i < NVMA; ++i) {
	    if(p->vma[i].used) {
	      memmove(&np->vma[i], &p->vma[i], sizeof(p->vma[i]));
	      filedup(p->vma[i].vfile);
	    }
	  }
	
	  safestrcpy(np->name, p->name, sizeof(p->name));
	  
	  ...
	}


####实验心得及总结
####学到的知识点
**mmap**
mmap是一种内存映射文件的方法，即将一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必再调用read,write等系统调用函数。相反，内核空间对这段区域的修改也直接反映用户空间，从而可以实现不同进程间的文件共享。   
**VMA**   
VM实现   
地址空间由VMA和页表组成   
VMA（虚拟内存区域）是一段连续的虚拟地址范围，这个地址范围内的权限相同(例如可读，可写等)，同时来自相同对象（文件，匿名内存）等    
VMA帮助内核决定如何处理页面错误

####mmap
1. 易错点：由于测试时会测试地址在栈空间之外等不合法的地方，因此产生读写中断时，需要首先判断地址是否合法。然后判断地址是否在某个文件映射的虚拟地址范围内，如果找到该文件，则读取磁盘，并将地址映射到产生中断的虚拟地址上。此外，还需要注意，由于一些地址并没有进行映射，因此在 walk 的时候，遇到这些地址直接跳过即可
2. 易错点：最后需要处理一些附带的问题： fork 和 exit 函数。在进程创建和退出时，需要复制和清空相应的文件映射，注意 exit 中取消映射代码的位置，如果放在后面会导致进程在 exit 时退不出来。
2. 感想：mmap实验也是虚拟内存的应用，刚开始觉得很难，不知道从何下手，因为要考虑的地方很多，比如映射的地址区域、如何组织管理VMA、映射的长度不为PGSIZE的倍数、父子进程需不需要共享物理内存等等，后来跟着lab的hint一步一步来，发现并没有那么复杂，很多都不需要考虑。
这次的实验中有的地方和实验五很相似，采用了懒加载。操作系统中很多需要较长时间和占用较大空间的操作都是能拖多久是多久，例如延迟分配物理内存、cow。 毫无疑问，这种策略带来了性能上的显著提升，特别是程序启动时间，同时也最大限度减少了内存的分配。


####实验结果
![](https://img-blog.csdnimg.cn/4a6abf1a997d4445a374a05fa3ef4485.png#pic_center)