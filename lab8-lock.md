# Lab8: Locks
[源码链接](https://github.com/babyseal1121/6.S081.git)
####实验介绍
**实验目的** 
 
在本实验室中，将重新设计xv6代码以提高并行性。在多核机器上，并行性差的一个常见症状是高强度的锁竞争。提高并行性通常需要改变数据结构和加锁策略，以减少争用。您将对 xv6 内存分配器和文件块缓存进行改进。 

**提示**  

	你可以使用kernel/param.h 中的NCPU （NCPU表示XV6使用了几个虚拟处理器，而xv6对于每个进程只有一个线程，该提示表示我们可以基于CPU数量进行修改）  

	让freerange 将所有空闲内存块给予 正在运行 freerange 的CPU（如果所有空闲内存都给了一个CPU，那么其他CPU怎么办？--从有空闲内存块的CPU拿吗？）  

	函数 cpuid() 返回当前的 cpu 编号， 不过需要关闭中断才能保证该函数被安全地使用。中断开关可使用 push_off() 和 pop_off()  

	用 kmem 命名你的锁 （提示我们需要创建额外的锁）

## Memory allocator

本实验完成的任务是为每个CPU都维护一个空闲列表，初始时将所有的空闲内存分配到某个CPU，此后各个CPU需要内存时，如果当前CPU的空闲列表上没有，则窃取其他CPU的。例如，所有的空闲内存初始分配到CPU0，当CPU1需要内存时就会窃取CPU0的，而使用完成后就挂在CPU1的空闲列表，此后CPU1再次需要内存时就可以从自己的空闲列表中取。

(1). 将`kmem`定义为一个数组，包含`NCPU`个元素，即每个CPU对应一个


	struct {
	  struct spinlock lock;
	  struct run *freelist;
	} kmem[NCPU];


(2). 修改`kinit`，为所有锁初始化以“kmem”开头的名称，该函数只会被一个CPU调用，`freerange`调用`kfree`将所有空闲内存挂在该CPU的空闲列表上


	void
	kinit()
	{
	  char lockname[8];
	  for(int i = 0;i < NCPU; i++) {
	    snprintf(lockname, sizeof(lockname), "kmem_%d", i);
	    initlock(&kmem[i].lock, lockname);
	  }
	  freerange(end, (void*)PHYSTOP);
	}


(3). 修改`kfree`，使用`cpuid()`和它返回的结果时必须关中断，请参考《XV6使用手册》第7.4节


	void
	kfree(void *pa)
	{
	  struct run *r;
	
	  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
	    panic("kfree");
	
	  // Fill with junk to catch dangling refs.
	  memset(pa, 1, PGSIZE);
	
	  r = (struct run*)pa;
	
	  push_off();  // 关中断
	  int id = cpuid();
	  acquire(&kmem[id].lock);
	  r->next = kmem[id].freelist;
	  kmem[id].freelist = r;
	  release(&kmem[id].lock);
	  pop_off();  //开中断
	}


(4). 修改`kalloc`，使得在当前CPU的空闲列表没有可分配内存时窃取其他内存的


	void *
	kalloc(void)
	{
	  struct run *r;
	
	  push_off();// 关中断
	  int id = cpuid();
	  acquire(&kmem[id].lock);
	  r = kmem[id].freelist;
	  if(r)
	    kmem[id].freelist = r->next;
	  else {
	    int antid;  // another id
	    // 遍历所有CPU的空闲列表
	    for(antid = 0; antid < NCPU; ++antid) {
	      if(antid == id)
	        continue;
	      acquire(&kmem[antid].lock);
	      r = kmem[antid].freelist;
	      if(r) {
	        kmem[antid].freelist = r->next;
	        release(&kmem[antid].lock);
	        break;
	      }
	      release(&kmem[antid].lock);
	    }
	  }
	  release(&kmem[id].lock);
	  pop_off();  //开中断
	
	  if(r)
	    memset((char*)r, 5, PGSIZE); // fill with junk
	  return (void*)r;
	}


# Buffer cache

这个实验的目的是将缓冲区的分配与回收并行化以提高效率，这个实验折腾了一天，有些内容还是比较绕的，

(1). 定义哈希桶结构，并在`bcache`中删除全局缓冲区链表，改为使用素数个散列桶


	#define NBUCKET 13
	#define HASH(id) (id % NBUCKET)
	
	struct hashbuf {
	  struct buf head;       // 头节点
	  struct spinlock lock;  // 锁
	};
	
	struct {
	  struct buf buf[NBUF];
	  struct hashbuf buckets[NBUCKET];  // 散列桶
	} bcache;
	```
	
	(2). 在`binit`中，（1）初始化散列桶的锁，（2）将所有散列桶的`head->prev`、`head->next`都指向自身表示为空，（3）将所有的缓冲区挂载到`bucket[0]`桶上，代码如下
	
	```c
	void
	binit(void) {
	  struct buf* b;
	  char lockname[16];
	
	  for(int i = 0; i < NBUCKET; ++i) {
	    // 初始化散列桶的自旋锁
	    snprintf(lockname, sizeof(lockname), "bcache_%d", i);
	    initlock(&bcache.buckets[i].lock, lockname);
	
	    // 初始化散列桶的头节点
	    bcache.buckets[i].head.prev = &bcache.buckets[i].head;
	    bcache.buckets[i].head.next = &bcache.buckets[i].head;
	  }
	
	  // Create linked list of buffers
	  for(b = bcache.buf; b < bcache.buf + NBUF; b++) {
	    // 利用头插法初始化缓冲区列表,全部放到散列桶0上
	    b->next = bcache.buckets[0].head.next;
	    b->prev = &bcache.buckets[0].head;
	    initsleeplock(&b->lock, "buffer");
	    bcache.buckets[0].head.next->prev = b;
	    bcache.buckets[0].head.next = b;
	  }
	}


(3). 在***buf.h***中增加新字段`timestamp`，这里来理解一下这个字段的用途：在原始方案中，每次`brelse`都将被释放的缓冲区挂载到链表头，禀明这个缓冲区最近刚刚被使用过，在`bget`中分配时从链表尾向前查找，这样符合条件的第一个就是最久未使用的。而在提示中建议使用时间戳作为LRU判定的法则，这样我们就无需在`brelse`中进行头插法更改结点位置


	struct buf {
	  ...
	  ...
	  uint timestamp;  // 时间戳
	};
	```
	
	(4). 更改`brelse`，不再获取全局锁
	
	```c
	void
	brelse(struct buf* b) {
	  if(!holdingsleep(&b->lock))
	    panic("brelse");
	
	  int bid = HASH(b->blockno);
	
	  releasesleep(&b->lock);
	
	  acquire(&bcache.buckets[bid].lock);
	  b->refcnt--;
	
	  // 更新时间戳
	  // 由于LRU改为使用时间戳判定，不再需要头插法
	  acquire(&tickslock);
	  b->timestamp = ticks;
	  release(&tickslock);
	
	  release(&bcache.buckets[bid].lock);
	}


(5). 更改`bget`，当没有找到指定的缓冲区时进行分配，分配方式是优先从当前列表遍历，找到一个没有引用且`timestamp`最小的缓冲区，如果没有就申请下一个桶的锁，并遍历该桶，找到后将该缓冲区从原来的桶移动到当前桶中，最多将所有桶都遍历完。在代码中要注意锁的释放


	static struct buf*
	bget(uint dev, uint blockno) {
	  struct buf* b;
	
	  int bid = HASH(blockno);
	  acquire(&bcache.buckets[bid].lock);
	
	  // Is the block already cached?
	  for(b = bcache.buckets[bid].head.next; b != &bcache.buckets[bid].head; b = b->next) {
	    if(b->dev == dev && b->blockno == blockno) {
	      b->refcnt++;
	
	      // 记录使用时间戳
	      acquire(&tickslock);
	      b->timestamp = ticks;
	      release(&tickslock);
	
	      release(&bcache.buckets[bid].lock);
	      acquiresleep(&b->lock);
	      return b;
	    }
	  }
	
	  // Not cached.
	  b = 0;
	  struct buf* tmp;
	
	  // Recycle the least recently used (LRU) unused buffer.
	  // 从当前散列桶开始查找
	  for(int i = bid, cycle = 0; cycle != NBUCKET; i = (i + 1) % NBUCKET) {
	    ++cycle;
	    // 如果遍历到当前散列桶，则不重新获取锁
	    if(i != bid) {
	      if(!holding(&bcache.buckets[i].lock))
	        acquire(&bcache.buckets[i].lock);
	      else
	        continue;
	    }
	
	    for(tmp = bcache.buckets[i].head.next; tmp != &bcache.buckets[i].head; tmp = tmp->next)
	      // 使用时间戳进行LRU算法，而不是根据结点在链表中的位置
	      if(tmp->refcnt == 0 && (b == 0 || tmp->timestamp < b->timestamp))
	        b = tmp;
	
	    if(b) {
	      // 如果是从其他散列桶窃取的，则将其以头插法插入到当前桶
	      if(i != bid) {
	        b->next->prev = b->prev;
	        b->prev->next = b->next;
	        release(&bcache.buckets[i].lock);
	
	        b->next = bcache.buckets[bid].head.next;
	        b->prev = &bcache.buckets[bid].head;
	        bcache.buckets[bid].head.next->prev = b;
	        bcache.buckets[bid].head.next = b;
	      }
	
	      b->dev = dev;
	      b->blockno = blockno;
	      b->valid = 0;
	      b->refcnt = 1;
	
	      acquire(&tickslock);
	      b->timestamp = ticks;
	      release(&tickslock);
	
	      release(&bcache.buckets[bid].lock);
	      acquiresleep(&b->lock);
	      return b;
	    } else {
	      // 在当前散列桶中未找到，则直接释放锁
	      if(i != bid)
	        release(&bcache.buckets[i].lock);
	    }
	  }
	
	  panic("bget: no buffers");
	}


(6). 最后将末尾的两个小函数也改一下


	void
	bpin(struct buf* b) {
	  int bid = HASH(b->blockno);
	  acquire(&bcache.buckets[bid].lock);
	  b->refcnt++;
	  release(&bcache.buckets[bid].lock);
	}
	
	void
	bunpin(struct buf* b) {
	  int bid = HASH(b->blockno);
	  acquire(&bcache.buckets[bid].lock);
	  b->refcnt--;
	  release(&bcache.buckets[bid].lock);
	}




####实验心得及总结
两个作业都是修改数据结构将粗粒度锁替换为细粒度锁降低锁争用增加并行度   
多线程问题往往不如单线程程序中的问题那样容易发现，并且需要对底层指令层面以及 CPU 运行原理层面有足够的认知，才能有效地发现并解决多线程问题。
####学到的知识点
**乐观锁**：乐观锁（optimistic locking）即在冲突发生概率很小的关键区内，不使用独占的互斥锁，而是在提交操作前，检查一下操作的数据是否被其他线程修改（在这里，检测的是 blockno 的缓存是否已被加入），如果是，则代表冲突发生，需要特殊处理（在这里的特殊处理即为直接返回已加入的 buf）。这样的设计，相比较「悲观锁（pessimistic locking）」而言，可以在冲突概率较低的场景下（例如 bget），降低锁开销以及不必要的线性化，提升并行性（例如在 bget 中允许「缓存是否存在」的判断并行化）。有时候还能用于避免死锁。

####Memory allocator
1. 易错点：在 initlock() 函数中, 锁名称的记录是指针的浅拷贝 lk->name=name, 因此对于每个锁的名称需要使用全局的内存进行记录而非函数的局部变量, 以防止内存丢失. 此外, 为了配合 kalloctest 的输出, 需要保证每个锁的名称以"kmem"开头.
2. 难点：偷取物理页的函数对我来说比较难，当前 CPU 的空闲物理页链表 freelist 为空, 但此时其他 CPU 可能仍有空闲物理页, 因此需要当前 CPU 去偷取其他 CPU 的部分物理页.   
首先考虑寻找仍有空闲物理页的 CPU 的方法, 此处便是选择的最为简单的循环遍历: 从当前 CPU 序号的下一个开始, 循环 NCPU-1 次, 即依次遍历剩余的 CPU 的空闲物理页链表, 直到找到一个链表不为空的.   
接下来考虑偷取物理页的数量, 指导书中仅说明为"部分", 此处选择的是偷取目标 CPU 一半的空闲物理页, 由于物理页是通过单向链表组织的, 因此此处采用了"快慢双指针"的算法来得到链表的中间结点, 将原链表一分为二, 后半部分作为目标 CPU 剩余的空闲物理页, 前半部分即偷取到的物理页.   
最后考虑加锁的问题.   
接下来考虑当前需要偷取的 CPU 的加锁情况, 根据分析可以看到, 在偷取时, 只是寻找其他 CPU 的空闲物理页, 对其他 CPU 的链表可能进行操作, 但不影响当前 CPU 的链表, 因此此时不能对当前 CPU 的空闲物理页链表加锁. 一旦加锁, 如若此时有另一个 CPU 同样要偷取其他 CPU 的物理页而遍历到当前 CPU 尝试获取其锁, 二者便会发生死锁.   
经过再次考虑, 会发现若不同时加锁也会有一定的问题, 但由于此处对于一个 CPU 而言不可能同时运行两个线程, 因此不会出现两个线程同时读取到同一 CPU 的空闲物理页为空然后同时去偷取其他 CPU 物理页致使的内存丢失情况.   
而最后分割获取到偷取到的空闲物理页, 对当前 CPU 的空闲物理页链表进行更新时, 再进行加锁.
####Buffer cache
1. 梳理一下几个锁的作用：拿着 bufmap_locks[key] 锁的时候，代表key桶这一个桶中的链表结构、以及所有链表节点的 refcnt 都不会被其他线程改变。也就是说，如果想访问/修改一个桶的结构，或者桶内任意节点的 refcnt，必须先拿那个桶 key 对应的 bufmap_locks[key] 锁。
（理由：1.只有 eviction 会改变某个桶的链表结构，而 eviction 本身也会尝试获取该锁 bufmap_locks[key]，所以只要占有该锁，涉及该桶的 eviction 就不会进行，也就代表该桶链表结构不会被改变；2.所有修改某个节点 refcnt 的操作都会先获取其对应的桶锁 bufmap_locks[key]，所以只要占有该锁，桶内所有节点的 refcnt 就不会改变。）
拿着 eviction_lock 的时候，代表不会有其他线程可以进行驱逐操作。由于只有 eviction 可以改变桶的链表结构，拿着该锁，也就意味着整个哈希表中的所有桶的链表结构都不会被改变，但不保证链表内节点的refcnt不会改变。也就是说，拿着 eviction_lock 的时候，refcnt 依然可能会因为多线程导致不一致，但是可以保证拿着锁的整个过程中，每个桶的链表节点数量不会增加、减少，也不会改变顺序。所以拿着 eviction_lock 的时候，可以安全遍历每个桶的每个节点，但是不能访问 refcnt。如果遍历的时候需要访问某个 buf 的 refcnt，则需要另外再拿其所在桶的 bufmap_locks[key] 锁。
2. 易错点：bget 方法一开始判断块是否在缓存中时也获取了一个桶的 bufmap_locks[key]，此时如果遍历获取所有桶的 bufmap_locks[i] 的话，很容易引起环路等待而触发死锁。若在判断是否存在后立刻释放掉 bufmap_locks[key] 再拿 eviction_lock 的话，又会导致在释放桶锁和拿 eviction_lock 这两个操作中间的微小间隙，其他线程可能会对同一个块号进行 bget 访问，导致最终同一个块被插入两次。   
最终方案是在释放 bufmap_locks[key]，获取 eviction_lock 之后，再判断一次目标块号是否已经插入。这意味着依然会出现 bget 尝试对同一个 blockno 进行驱逐并插入两次的情况，但是能够保证除了第一个驱逐+插入的尝试能成功外，后续的尝试都不会导致重复驱逐+重复插入，而是能正确返回第一个成功的驱逐+插入产生的结果。

####实验结果
![](https://img-blog.csdnimg.cn/78d6752c7899476fbe8dc6b670f80b31.png#pic_center)