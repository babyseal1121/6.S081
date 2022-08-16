# Lab9: file system
[源码链接](https://github.com/babyseal1121/6.S081.git)

## Large files
####实验目的
xv6中的 inode 有 12个直接索引（直接对应了 data 区域的磁盘块），1个一级索引（存放另一个指向 data 区域的索引）。因此，最多支持 12 + 256 = 268 个数据块。  
现在实现要求变为11个直接索引，1个一级间接索引，1个二级间接索引，由上述推理可知，1个二级间接索引可以容纳256 * 256 = 65536个块地址，因此现在文件可达的最大大小变为11 + 256 + 65536 = 65803个数据块。
**提示**  

	新增一个二级目录以支持（11 + 256 + 256*256）= 65803大小的文件
	fs.h 中的 struct dinode,NDIRECT,NINDIRECT,MAXFILE 是你需要关注的内容
	修改后你的inode结构应该大致如下：addrs[] 中，前十一项为磁盘块，第十二项为一级目录，第十三项为二级目录
	如果你修改了NDIRECT ，你需要修改 file.h/struct inode/addr[], 确保struct inode 和 struct dinode 的 addrs[] 的大小相同（这里一定要修改为11，不然内核会把你的第12块也识别为数据磁盘块）
	确保在每次 bread() 后 调用 brelse()
	在需要时才给一级目录和二级目录中分配磁盘块

####实验步骤
(1). 在fs.h中添加宏定义


	#define NDIRECT 11
	#define NINDIRECT (BSIZE / sizeof(uint))
	#define NDINDIRECT ((BSIZE / sizeof(uint)) * (BSIZE / sizeof(uint)))
	#define MAXFILE (NDIRECT + NINDIRECT + NDINDIRECT)
	#define NADDR_PER_BLOCK (BSIZE / sizeof(uint))  // 一个块中的地址数量


(2). 由于`NDIRECT`定义改变，其中一个直接块变为了二级间接块，需要修改inode结构体中`addrs`元素数量

	// fs.h
	struct dinode {
	  ...
	  uint addrs[NDIRECT + 2];   // Data block addresses
	};
	
	// file.h
	struct inode {
	  ...
	  uint addrs[NDIRECT + 2];
	};

(3). 修改`bmap`支持二级索引


	static uint
	bmap(struct inode *ip, uint bn)
	{
	  uint addr, *a;
	  struct buf *bp;
	
	  if(bn < NDIRECT){
	    ...
	  }
	  bn -= NDIRECT;
	
	  if(bn < NINDIRECT){
	    ...
	  }
	  bn -= NINDIRECT;
	
	  // 二级间接块的情况
	  if(bn < NDINDIRECT) {
	    int level2_idx = bn / NADDR_PER_BLOCK;  // 要查找的块号位于二级间接块中的位置
	    int level1_idx = bn % NADDR_PER_BLOCK;  // 要查找的块号位于一级间接块中的位置
	    // 读出二级间接块
	    if((addr = ip->addrs[NDIRECT + 1]) == 0)
	      ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
	    bp = bread(ip->dev, addr);
	    a = (uint*)bp->data;
	
	    if((addr = a[level2_idx]) == 0) {
	      a[level2_idx] = addr = balloc(ip->dev);
	      // 更改了当前块的内容，标记以供后续写回磁盘
	      log_write(bp);
	    }
	    brelse(bp);
	
	    bp = bread(ip->dev, addr);
	    a = (uint*)bp->data;
	    if((addr = a[level1_idx]) == 0) {
	      a[level1_idx] = addr = balloc(ip->dev);
	      log_write(bp);
	    }
	    brelse(bp);
	    return addr;
	  }
	
	  panic("bmap: out of range");
	}


(4). 修改`itrunc`释放所有块


	void
	itrunc(struct inode *ip)
	{
	  int i, j;
	  struct buf *bp;
	  uint *a;
	
	  for(i = 0; i < NDIRECT; i++){
	    ...
	  }
	
	  if(ip->addrs[NDIRECT]){
	    ...
	  }
	
	  struct buf* bp1;
	  uint* a1;
	  if(ip->addrs[NDIRECT + 1]) {
	    bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
	    a = (uint*)bp->data;
	    for(i = 0; i < NADDR_PER_BLOCK; i++) {
	      // 每个一级间接块的操作都类似于上面的
	      // if(ip->addrs[NDIRECT])中的内容
	      if(a[i]) {
	        bp1 = bread(ip->dev, a[i]);
	        a1 = (uint*)bp1->data;
	        for(j = 0; j < NADDR_PER_BLOCK; j++) {
	          if(a1[j])
	            bfree(ip->dev, a1[j]);
	        }
	        brelse(bp1);
	        bfree(ip->dev, a[i]);
	      }
	    }
	    brelse(bp);
	    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
	    ip->addrs[NDIRECT + 1] = 0;
	  }
	
	  ip->size = 0;
	  iupdate(ip);
	}


## Symbolic links
####实验目的
硬链接是指多个文件名指向同一个inode号码。有以下特点：

可以用不同的文件名访问同样的内容；
对文件内容进行修改，会影响到所有文件名；
删除一个文件名，不影响另一个文件名的访问。
而软链接也是一个文件，但是文件内容指向另一个文件的 inode。打开这个文件时，会自动打开它指向的文件，类似于 windows 系统的快捷方式。

xv6 中没有符号链接（软链接），这个任务需要我们实现一个符号链接。  

**提示**

	在 kernel/stat.h 中添加一个新的文件类型 T_SYMLINK
	在kernel/fcntl.h中添加一个新标志O_NOFOLLOW，该标志可用于开放系统调用。请注意，传递给open的标志使用位OR运算符组合，因此新标志不应与任何现有标志重叠。
	在symlink中，你需要选择一个地方储存你的目标路径，比如在inode 的数据块里
	修改open() 来增加处理软链接的情况
	允许递归软链接的情况，但需要设置一个递归深度上限
####实验步骤

(1). 配置系统调用的常规操作，如在***user/usys.pl***、***user/user.h***中添加一个条目，在***kernel/syscall.c***、***kernel/syscall.h***中添加相关内容

(2). 添加提示中的相关定义，`T_SYMLINK`以及`O_NOFOLLOW`


	// fcntl.h
	#define O_NOFOLLOW 0x004
	// stat.h
	#define T_SYMLINK 4


(3). 在***kernel/sysfile.c***中实现`sys_symlink`，这里需要注意的是`create`返回已加锁的inode，此外`iunlockput`既对inode解锁，还将其引用计数减1，计数为0时回收此inode


	uint64
	sys_symlink(void) {
	  char target[MAXPATH], path[MAXPATH];
	  struct inode* ip_path;
	
	  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0) {
	    return -1;
	  }
	
	  begin_op();
	  // 分配一个inode结点，create返回锁定的inode
	  ip_path = create(path, T_SYMLINK, 0, 0);
	  if(ip_path == 0) {
	    end_op();
	    return -1;
	  }
	  // 向inode数据块中写入target路径
	  if(writei(ip_path, 0, (uint64)target, 0, MAXPATH) < MAXPATH) {
	    iunlockput(ip_path);
	    end_op();
	    return -1;
	  }
	
	  iunlockput(ip_path);
	  end_op();
	  return 0;
	}


(4). 修改`sys_open`支持打开符号链接


	uint64
	sys_open(void)
	{
	  ...
	  
	  if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
	    ...
	  }
	
	  // 处理符号链接
	  if(ip->type == T_SYMLINK && !(omode & O_NOFOLLOW)) {
	    // 若符号链接指向的仍然是符号链接，则递归的跟随它
	    // 直到找到真正指向的文件
	    // 但深度不能超过MAX_SYMLINK_DEPTH
	    for(int i = 0; i < MAX_SYMLINK_DEPTH; ++i) {
	      // 读出符号链接指向的路径
	      if(readi(ip, 0, (uint64)path, 0, MAXPATH) != MAXPATH) {
	        iunlockput(ip);
	        end_op();
	        return -1;
	      }
	      iunlockput(ip);
	      ip = namei(path);
	      if(ip == 0) {
	        end_op();
	        return -1;
	      }
	      ilock(ip);
	      if(ip->type != T_SYMLINK)
	        break;
	    }
	    // 超过最大允许深度后仍然为符号链接，则返回错误
	    if(ip->type == T_SYMLINK) {
	      iunlockput(ip);
	      end_op();
	      return -1;
	    }
	  }
	
	  if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
	    ...
	  }
	
	  ...
	  return fd;
	}

####实验心得及总结
####学到的知识点
**文件系统**
xv6 的文件系统和许多其他系统的实现大体是相同的，只不过很多地方简化了很多。存储文件的方式都是以 block 的形式。物理磁盘在读写时，是以扇区为单位的，通常来讲，每个扇区是 512 个字节。但操作系统读取磁盘时，由于寻道的时间是很长的，读写数据的时间反而没那么久，因此会操作系统一般会读写连续的多个扇区，所使用的时间几乎一样。操作系统以多个扇区作为一个磁盘块，xv6是两个扇区，即一个 block 为 1024 个字节。  
磁盘只是以扇区的形式存储数据，但如果没有一个读取的标准，该磁盘就是一个生磁盘。磁盘需要按照操作系统读写的标准来存储数据，格式如下：
![](https://pic3.zhimg.com/80/v2-656c90bac78a76fabb611a19d0e92832_720w.jpg)  
从中可以看到，磁盘中不同区域的数据块有不同的功能。第 0 块数据块是启动区域，计算机启动就是从这里开始的；第 1 块数据是超级块，存储了整个磁盘的信息；然后是 log 区域，用于故障恢复；bit map 用于标记磁盘块是否使用；然后是 inode 区域 和 data 区域。  
磁盘中主要存储文件的 block 是 inode 和 data。操作系统中，文件的信息是存放在 inode 中的，每个文件对应了一个 inode，inode 中含有存放文件内容的磁盘块的索引信息，用户可以通过这些信息来查找到文件存放在磁盘的哪些块中。inodes 块中存储了很多文件的 inode。
####Large files
看bmap() 寻址操作，就是读取磁盘块上的数据到 struct buf 然后使用buf->data 进行下标寻址，和前十一项一样。
可以推测二级目录则是再进行一次一级目录索引的过程。但是需要修改下标。这里注意在一级索引的基础上修改二级索引要求仔细一点，不然很容易出错。
####Symbolic links
1. 在跟踪符号链接时需要额外考虑到符号链接的目标可能还是符号链接, 此时需要递归的去跟踪目标链接直至得到真正的文件. 而这其中需要解决两个问题: 一是符号链接可能成环, 这样会一直递归地跟踪下去, 因此需要进行成环的检测; 另一方面是需要对链接的深度进行限制, 以减轻系统负担
2. 难点：成环检测。   
记录每次跟踪到的文件的 inode number, 每次跟踪到目标文件的 inode 后都会将其 inode number 与 inums 数组中记录的进行比较, 若有重复则证明成环.   
因此整体上 follow_symlink() 函数的流程其实比较简单, 就是至多循环 NSYMLINK 次去递归的跟踪符号链接: 使用 readi() 函数读出符号链接文件中记录的目标文件的路径, 然后使用 namei() 获取文件路径对应的 inode, 然后与已经记录的 inode number 比较判断是否成环. 直到目标 inode 的类型不为 T_SYMLINK 即符号链接类型.
而在 sys_open() 中, 需要在创建文件对象 f=filealloc() 之前, 对于符号链接, 在非 NO_FOLLOW 的情况下需要将当前文件的 inode 替换为由 follow_symlink() 得到的目标文件的 inode 再进行后续的操作.
最后考虑这个过程中的加锁释放的规则. 对于原本不考虑符号链接的情况, 在 sys_open() 中, 由 create() 或 namei() 获取了当前文件的 inode 后实际上就持有了该 inode 的锁, 直到函数结束才会通过 iunlock() 释放(当执行成功时未使用 iput() 释放 inode 的 ref 引用, 笔者认为后续到 sys_close() 调用执行前该 inode 一直会处于活跃状态用于对该文件的操作, 因此不能减少引用). 而对于符号链接, 由于最终打开的是链接的目标文件, 因此一定会释放当前 inode 的锁转而获取目标 inode 的锁. 而在处理符号链接时需要对 ip->type 字段进行读取, 自然此时也不能释放 inode 的锁, 因此在进入 follow_symlink() 时一直保持着 inode 的锁的持有, 当使用 readi() 读取了符号链接中记录的目标文件路径后, 此时便不再需要当前符号链接的 inode, 便使用 iunlockput() 释放锁和 inode. 当在对目标文件的类型判断是否不为符号链接时, 此时再对其进行加锁. 这样该函数正确返回时也会持有目标文件 inode 的锁, 达到了函数调用前后的一致.

####感想
和前面几个实验不一样的是，这个实验主要针对文件系统的设计。文件系统的知识有点忘了，主要还是多看看其他文件系统函数是如何实现的（sys_open 函数帮助很大，还有 namei, readi, writei 等跟 inode 有关的函数）。
####实验结果
![](https://img-blog.csdnimg.cn/f65ffaa22d09497d8a6c666e2e301803.png#pic_center)