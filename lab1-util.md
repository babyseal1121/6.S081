# lab1: Util

[源码链接](https://github.com/babyseal1121/6.S081.git)
##实验目的：
熟悉xv6及其系统调用
##实验内容
## sleep
####实验要求
为 xv6 系统实现 UNIX 的 sleep 程序。你的 sleep 程序应该使当前进程暂停相应的时钟周期数，时钟周期数由用户指定。解决方案应该在文件 user/sleep.c中。
####实验思路及步骤
（1）根据实验给的提示信息，找到所需的头文件并引入，即 kernel/types.h 声明类型的头文件和 user/user.h 声明系统调用函数和 ulib.c 中函数的头文件。

（2）判断用户输入的参数是否正确，只要判断命令行参数不等于 2 个，就可以知道用户输入的参数有误，就可以打印错误信息。而main(int argc,char* argv[]) 函数中，参数 argc 是命令行总参数的个数，参数 argv[] 是 argc 个参数，其中第 0 个参数是程序的全名，其他的参数是命令行后面跟的用户输入的参数。即判断argc是否不等于2。

（3）如果argc不等于2的话，说明输入参数有误，输出错误信息，并退出程序，返回1表示异常；如果等于2的话，说明输入参数无误，获取参数并转为整数型（用atoi() 函数）。

（4）调用系统调用 sleep 函数，传入参数，最后调用系统调用 exit(0) 使程序正常退出。从而得到了3.1.3的代码，将代码存在文件 user/sleep.c中。

（5）在 Makefile 文件中添加配置，在 UPROGS 项中最后一行添加 $U/_sleep\ 。

（6）输入命令 make qemu进行结果检验



	int main(int argc, char const *argv[])
	{
  		if (argc != 2) { //参数错误
    	fprintf(2, "usage: sleep <time>\n");
    	exit(1);
  	}
  	sleep(atoi(argv[1]));
  	exit(0);
	}



## pingpong

使用两个管道进行父子进程通信，需要注意的是如果管道的写端没有`close`，那么管道中数据为空时对管道的读取将会阻塞。因此对于不需要的管道描述符，要尽可能早的关闭。
####实验要求
使用 UNIX 系统调用编写一个程序 pingpong ，在一对管道上实现两个进程之间的通信。父进程应该通过第一个管道给子进程发送一个信息 “ping”，子进程接收父进程的信息后打印 "<pid>: received ping" ，其中<pid>是其进程 ID 。然后子进程通过另一个管道发送一个信息 “pong” 给父进程，父进程接收子进程的信息然后打印 "<pid>: received pong" ，然后退出。解决方案应该在文件 user/pingpong.c中。
####实验思路及步骤

（1）根据实验提示引入所需要的头文件：kernel/types.h，user/user.h和stddef.h。

（2）父进程与子进程之间需要两个管道传递信息，因此定义两个数组ptoc_fd和ctop_fd，一个用于父进程给子进程传递信息，另一个用于子进程给父进程传递信息。

（3）调用pipe(pipefd)系统调用创建管道，pipefd[0]（即上面的ptoc_fd[0]和ctop_fd[0]）表示管道的读取端，pipefd[1]（即上面的ptoc_fd[1]和ctop_fd[1]）表示管道的写入端。

（4）创建缓冲区字符数组buf（本实验中buf的大小为8），用于存放父进程与子进程之间传递的信息。

（5）调用fork创建子进程，如果fork()返回值为0，说明是子进程，否则是父进程（因为父进程的fork返回的是子进程的pid），故而可以凭此来判断子、父进程，从而编写子、父进程的代码。

（6）子进程：读取父进程传来的信息->打印相关信息->写入传给父进程的信息

（7）父进程：写入传给子进程的信息->等待子进程读取并写入信息（即子进程执行完）->读取子进程传来的信息->打印相关信息。其中，可以调用getpid()获取进程的pid。

（8）由上可以得到3.2.3的代码，将代码存在文件 user/pingpong.c中。

（9）在 Makefile 文件中添加配置，在 UPROGS 项中最后一行添加 $U/_pingpong\ 。

（10）输入命令 make qemu进行结果检验，检验完后Ctrl+a -x退出qemu，输入./grade-lab-util pingpong进行测试。


	

	#define RD 0 //pipe的read端
	#define WR 1 //pipe的write端

	int main(int argc, char const *argv[]) {
    char buf = 'P'; //用于传送的字节

    int fd_c2p[2]; //子进程->父进程
    int fd_p2c[2]; //父进程->子进程
    pipe(fd_c2p);
    pipe(fd_p2c);

    int pid = fork();
    int exit_status = 0;

    if (pid < 0) {
        fprintf(2, "fork() error!\n");
        close(fd_c2p[RD]);
        close(fd_c2p[WR]);
        close(fd_p2c[RD]);
        close(fd_p2c[WR]);
        exit(1);
    } else if (pid == 0) { //子进程
        close(fd_p2c[WR]);
        close(fd_c2p[RD]);

        if (read(fd_p2c[RD], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "child read() error!\n");
            exit_status = 1; //标记出错
        } else {
            fprintf(1, "%d: received ping\n", getpid());
        }

        if (write(fd_c2p[WR], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "child write() error!\n");
            exit_status = 1;
        }

        close(fd_p2c[RD]);
        close(fd_c2p[WR]);

        exit(exit_status);
    } else { //父进程
        close(fd_p2c[RD]);
        close(fd_c2p[WR]);

        if (write(fd_p2c[WR], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "parent write() error!\n");
            exit_status = 1;
        }

        if (read(fd_c2p[RD], &buf, sizeof(char)) != sizeof(char)) {
            fprintf(2, "parent read() error!\n");
            exit_status = 1; //标记出错
        } else {
            fprintf(1, "%d: received pong\n", getpid());
        }

        close(fd_p2c[WR]);
        close(fd_c2p[RD]);

        exit(exit_status);
    }
}


## primes

它的思想是多进程版本的递归，不断地将左邻居管道中的数据筛选后传送给右邻居，每次传送的第一个数据都将是一个素数。
####实验要求
使用管道将2至35中的素数筛选出来，这个想法归功于Unix 管道的发明者Doug McIlroy。目标是使用pipe和fork来设置管道。第一个过程将数字2到35输入管道。对于每个素数，您将安排创建一个进程，该进程通过管道从其左邻居读取并通过另一管道向其右邻居写入。由于xv6的文件描述符和进程数量有限，第一个进程可以在35处停止。解决方案应该放在 user/primes.c 文件中。
####实验思路及步骤
（1）首先要将2-35写入管道中，然后选出其中的素数并打印。该步的方法与上个小实验类似，不过是用子进程来写入2-35，等待子进程执行完后，父进程来实现选出素数并打印的功能。

（2）求素数：2是素数，直接打印，将剩下的数中是2的倍数的数剔除（即将不是2的倍数的数写入另一个管道）；剩下的数中第一个数是3，是素数，直接打印，把剩下的数中不是3的倍数的数剔除，依此类推。可以发现每一次操作第一个数一定是素数（因为没有被剔除说明不能被任何小于它的数整除），直接打印，将剩下的数中是第一个数的倍数的数剔除，写入另一个管道，等待下一轮检验。此过程可以用递归实现，涉及到子父进程，子进程用来筛选，父进程用来进入下一轮（即递归）（因为只能筛选完后才能进入下一轮）。

（3）根据实验提示第一条：请关闭进程不需要的文件描述符，否则程序将在第一个进程到达 35 之前耗尽 xv6 资源。因此需要文件描述符重定向，关闭进程不需要的文件描述符。由上可以得到3.3.3的实验代码，将其保存在user/primes.c 文件中，并在 Makefile 文件中添加配置，在 UPROGS 项中最后一行添加 $U/_primes\ 。

（4）输入命令 make qemu进行结果检验，检验完后Ctrl+a -x退出qemu，输入./grade-lab-util primes进行测试。

```c
	
	#define RD 0
	#define WR 1

	const uint INT_LEN = sizeof(int);

	/**
 	* @brief 读取左邻居的第一个数据
 	* @param lpipe 左邻居的管道符
	 * @param pfirst 用于存储第一个数据的地址
 	* @return 如果没有数据返回-1,有数据返回0
 	*/
	int lpipe_first_data(int lpipe[2], int *dst)
	{
  	if (read(lpipe[RD], dst, sizeof(int)) == sizeof(int)) {
    	printf("prime %d\n", *dst);
    	return 0;
  	}
  	return -1;
	}

	/**
	 * @brief 读取左邻居的数据，将不能被first整除的写入右邻居
 	* @param lpipe 左邻居的管道符
 	* @param rpipe 右邻居的管道符
 	* @param first 左邻居的第一个数据
 	*/
	void transmit_data(int lpipe[2], int rpipe[2], int first)
	{
  	int data;
  	// 从左管道读取数据
  	while (read(lpipe[RD], &data, sizeof(int)) == sizeof(int)) {
   	 // 将无法整除的数据传递入右管道
    	if (data % first)
      	write(rpipe[WR], &data, sizeof(int));
  	}
  	close(lpipe[RD]);
  	close(rpipe[WR]);
	}

	/**
	 * @brief 寻找素数
 	* @param lpipe 左邻居管道
 	*/
	void primes(int lpipe[2])
	{
  	close(lpipe[WR]);
  	int first;
  	if (lpipe_first_data(lpipe, &first) == 0) {
    	int p[2];
    	pipe(p); // 当前的管道
    transmit_data(lpipe, p, first);

    if (fork() == 0) {
      primes(p);    // 递归的思想，但这将在一个新的进程中调用
    } else {
      close(p[RD]);
      wait(0);
    }
  	}
  	exit(0);
	}

	int main(int argc, char const *argv[])
	{
  	int p[2];
  	pipe(p);

  	for (int i = 2; i <= 35; ++i) //写入初始数据
    	write(p[WR], &i, INT_LEN);

  	if (fork() == 0) {
   	 primes(p);
  	} else {
    	close(p[WR]);
    	close(p[RD]);
    	wait(0);
  	}

  	exit(0);
	}
```

## find
####实验要求
编写一个简单的 UNIX find 程序，在目录树中查找包含特定名称的所有文件。解决方案应放在user/find.c 文件中。
####实验思路及步骤
（1）可以参考利用user/ls.c中的大部分代码，首先判断参数个数是否符合要求（即参数个数是否为3），如果符合，则调用find函数查找相关目录下的文件；如果不符合，则输出错误信息，退出并返回1。

（2）find函数：第一个参数为目录，第二个参数为文件名。首先，声明所需要的文件名缓冲区、文件描述符、与文件相关的结构体（dirent和stat）。然后测试是否能进入给定路径，测试获得的存在的文件模式是否为目录。如果测试都通过则将路径拷贝，循环获取路径下的文件名，并与要查找的文件名进行比较，如果是文件且与要查找文件名相同则输出路径，如果是目录则递归调用find()函数，其中，根据提示不要递归”.”和”..”。

（3）由上可以得到3.4.3的代码，将代码保存到user/find.c文件中，并在 Makefile 文件中添加配置，在 UPROGS 项中最后一行添加 $U/_find\ 。

（4）输入命令 make qemu进行结果检验，检验完后Ctrl+a -x退出qemu，输入./grade-lab-util find进行测试。

```c

	void find(char *path, const char *filename)
	{
 	char buf[512], *p;
  	int fd;
  	struct dirent de;
  	struct stat st;

  	if ((fd = open(path, 0)) < 0) {
    	fprintf(2, "find: cannot open %s\n", path);
    	return;
  	}

  	if (fstat(fd, &st) < 0) {
    	fprintf(2, "find: cannot fstat %s\n", path);
    	close(fd);
    	return;
  	}

  	//参数错误，find的第一个参数必须是目录
  	if (st.type != T_DIR) {
    	fprintf(2, "usage: find <DIRECTORY> <filename>\n");
    	return;
  	}

 	 if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf) {
    	fprintf(2, "find: path too long\n");
    	return;
  	}
  	strcpy(buf, path);
  	p = buf + strlen(buf);
  	*p++ = '/'; //p指向最后一个'/'之后
  	while (read(fd, &de, sizeof de) == sizeof de) {
    	if (de.inum == 0)
      continue;
    	memmove(p, de.name, DIRSIZ); //添加路径名称
    	p[DIRSIZ] = 0;               //字符串结束标志
    	if (stat(buf, &st) < 0) {
      	fprintf(2, "find: cannot stat %s\n", buf);
      	continue;
    	}
    	//不要在“.”和“..”目录中递归
    	if (st.type == T_DIR && strcmp(p, ".") != 0 && strcmp(p, "..") != 0) {
      	find(buf, filename);
    	} else if (strcmp(filename, p) == 0)
      	printf("%s\n", buf);
  	}

  	close(fd);
	}

	int main(int argc, char *argv[])
	{
  	if (argc != 3) {
   		fprintf(2, "usage: find <directory> <filename>\n");
    	exit(1);
  	}
  	find(argv[1], argv[2]);
  	exit(0);
	}
```

## xargs
####实验要求
编写一个简单版本的 UNIX xargs 程序：从标准输入读取行并为每一行运行一个命令，将该行作为参数提供给命令。解决方案应该在user/xargs.c文件中。
####实验思路及步骤
（1）首先判断参数个数，如果小于2说明参数输入不正确，输出错误提示信息，退出并返回1。

（2）定义一个数组用于存放子进程的参数，大小为MAXARG（MAXARG在kernel/param.h中有定义），创建索引和缓冲区。

（3）然后，循环读取管道中的数据，放入缓冲区，建立一个新的临时缓冲区存放追加的参数。把临时缓冲区追加到子进程参数列表后面。并循环获取缓冲区字符，当该字符不是换行符时，直接给临时缓冲区；否则创建一个子进程，把执行的命令和参数列表传入 exec() 函数中，执行命令。

（4）由上可以得到的代码，将其保存在user/xargs.c文件中，并在 Makefile 文件中添加配置，在 UPROGS 项中最后一行添加 $U/_xargs.c\ 。

（5）输入命令 make qemu进行结果检验，检验完后Ctrl+a -x退出qemu，输入./grade-lab-util xargs.c进行测试。
```c
	

	define MAXSZ 512
	// 有限状态自动机状态定义
	enum state {
  	S_WAIT,         // 等待参数输入，此状态为初始状态或当前字符为空格
  	S_ARG,          // 参数内
  	S_ARG_END,      // 参数结束
  	S_ARG_LINE_END, // 左侧有参数的换行，例如"arg\n"
  	S_LINE_END,     // 左侧为空格的换行，例如"arg  \n""
  	S_END           // 结束，EOF
	};
	
	// 字符类型定义
	enum char_type {
 	 C_SPACE,
  	C_CHAR,
  	C_LINE_END
	};

	/**
 	* @brief 获取字符类型
 	*
 	* @param c 待判定的字符
 	* @return enum char_type 字符类型
 	*/
	enum char_type get_char_type(char c)
	{
  	switch (c) {
  	case ' ':
    	return C_SPACE;
  	case '\n':
    	return C_LINE_END;
  	default:
    	return C_CHAR;
  	}
	}

	/**
 	* @brief 状态转换
 	*
 	* @param cur 当前的状态
 	* @param ct 将要读取的字符
 	* @return enum state 转换后的状态
 	*/
	enum state transform_state(enum state cur, enum char_type ct)
	{
  	switch (cur) {
  	case S_WAIT:
   	 	if (ct == C_SPACE)    return S_WAIT;
   	 	if (ct == C_LINE_END) return S_LINE_END;
    	if (ct == C_CHAR)     return S_ARG;
    	break;
  	case S_ARG:
    	if (ct == C_SPACE)    return S_ARG_END;
    	if (ct == C_LINE_END) return S_ARG_LINE_END;
    	if (ct == C_CHAR)     return S_ARG;
    	break;
  	case S_ARG_END:
  	case S_ARG_LINE_END:
  	case S_LINE_END:
    	if (ct == C_SPACE)    return S_WAIT;
   	 	if (ct == C_LINE_END) return S_LINE_END;
    	if (ct == C_CHAR)     return S_ARG;
    	break;
  	default:
   	 break;
  		}
  		return S_END;
	}


	/**
 	* @brief 将参数列表后面的元素全部置为空
 		*        用于换行时，重新赋予参数
 	*
 	* @param x_argv 参数指针数组
 	* @param beg 要清空的起始下标
 	*/
	void clearArgv(char *x_argv[MAXARG], int beg)
	{
  	for (int i = beg; i < MAXARG; ++i)
    x_argv[i] = 0;
	}

	int main(int argc, char *argv[])
	{
  	if (argc - 1 >= MAXARG) {
   	 fprintf(2, "xargs: too many arguments.\n");
    	exit(1);
  	}
  	char lines[MAXSZ];
  	char *p = lines;
  	char *x_argv[MAXARG] = {0}; // 参数指针数组，全部初始化为空指针

  	// 存储原有的参数
  	for (int i = 1; i < argc; ++i) {
    x_argv[i - 1] = argv[i];
  	}
  	int arg_beg = 0;          // 参数起始下标
  	int arg_end = 0;          // 参数结束下标
  	int arg_cnt = argc - 1;   // 当前参数索引
  	enum state st = S_WAIT;   // 起始状态置为S_WAIT

  	while (st != S_END) {
    // 读取为空则退出
    if (read(0, p, sizeof(char)) != sizeof(char)) {
      st = S_END;
    } else {
      st = transform_state(st, get_char_type(*p));
    }

    if (++arg_end >= MAXSZ) {
      fprintf(2, "xargs: arguments too long.\n");
      exit(1);
    }

    switch (st) {
    case S_WAIT:          // 这种情况下只需要让参数起始指针前移
      ++arg_beg;
      break;
    case S_ARG_END:       // 参数结束，将参数地址存入x_argv数组中
      x_argv[arg_cnt++] = &lines[arg_beg];
      arg_beg = arg_end;
      *p = '\0';          // 替换为字符串结束符
      break;
    case S_ARG_LINE_END:  // 将参数地址存入x_argv数组中同时执行指令
      x_argv[arg_cnt++] = &lines[arg_beg];
      // 不加break，因为后续处理同S_LINE_END
    case S_LINE_END:      // 行结束，则为当前行执行指令
      arg_beg = arg_end;
      *p = '\0';
      if (fork() == 0) {
        exec(argv[1], x_argv);
      }
      arg_cnt = argc - 1;
      clearArgv(x_argv, arg_cnt);
      wait(0);
      break;
    default:
      break;
    }

    ++p;    // 下一个字符的存储位置后移
  	}
  	exit(0);
	}
```
##实验心得体会
####sleep
1. sleep：第一个实验让我们利用系统调用，来实现用户态调用sleep函数。sleep功能为使进程睡眠若干个时钟周期（xv6中一个tick为100ms），首先创建user/sleep.c源文件，引入user.h头文件，系统调用和工具函数都定义在该文件里。
2. 拿到题目一看，以为会很难，是让自己写这个sleep的系统调用，后来一看user/user.h，发现这个系统调用已经被实现了，自己的代码里面直接调user/user.h 里面的 sleep就行。
3. 头文件和main函数的写法参考其他的 user 目录下的 .c 文件即可。
4. 在实现功能的时候，还需要判断一下命令行的输入是否正确。通过阅读其他user文件夹中的源码，学习到实现方式：若输入的内容超过两项，则打印输入错误信息。
####pingpong
2. 使用 pipe() 和 fork() 实现父进程发送一个字符，子进程成功接收该字符后打印 received ping，再向父进程发送一个字符，父进程成功接收后打印 received pong。这里需要懂得什么是文件描述符。在 Linux 操作系统中，一切皆文件，内核通过文件描述符访问文件。
3. 每个进程都会默认打开3个文件描述符,即0、1、2。其中0代表标准输入流（stdin）、1代表标准输出流（stdout）、2代表标准错误流（stderr）。可以使用 > 或 >> 的方式重定向。
4. 管道是操作系统进程间通信的一种方式，可以使用 pipe() 函数进行创建。有在管道创建后，就有两个文件描述符，一个是负责读，一个是负责写。两个描述符对应内核中的同一块内存，管道的主要特点：数据在管道中以字节流的形式传输，由于管道是半双工的，管道中的数据只能单向流动。
管道只能用于具有亲缘关系的进程通信，父子进程或者兄弟进程。
在 fork() 后，父进程和子进程拥有相同的管道描述符，通常为了安全起见，我们关闭一个进程的读描述符和另一个进程的写描述符，这样就可以让数据更安全地传输。fork调用的一个奇妙之处就是它仅仅被调用一次，却能够返回两次，它可能有三种不同的返回值：
    1）在父进程中，fork返回新创建子进程的进程ID；
    2）在子进程中，fork返回0；
    3）如果出现错误，fork返回一个负值；
5. 在实现时首先新建文件描述符数组和缓冲区，之后利用pipe()系统调用建立两个管道。在完成功能时，首先根据fork()函数判断是父进程还是子进程，然后完成相关的读写操作。
6. 在网络上看到一张帮助理解的图片：
![](https://pic4.zhimg.com/80/v2-9a3736c4a8b2a32cc805d95004d710cf_720w.jpg)
####prime
1. 任务是输出区间内的质数 [2, 35]，如果一般的方法，直接挨个遍历计算就行，但由于这里需要充分使用 pipe 和 fork 方法，因此需要一些不一样的方法。
2. 下面使用的方法是埃拉托斯特尼素数筛，简称筛法。简单地说就是，每次得到一个素数时，在所有小于 n 的数中，删除它的倍数，然后不断迭代，剩下的就全是素数了。
3. 实现思想是，只要还没获取到所有的素数，便不断遍历/递归。每次递归时： 1. 先在父进程中创建一个子进程。 2.利用子进程将剩下的所有数全都写到管道中。 3. 在父进程中，将数不断读出来，管道中第一个数一定是素数，然后删除它的倍数（如果不是它的倍数，就继续更新数组，同时记录数组中目前已有的数字数量）。
4. 在写代码时需要注意的两个点：1.文件描述符溢出： xv6限制fd的范围为0~15，而每次pipe()都会创建两个新的fd，如果不及时关闭不需要的fd，会导致文件描述符资源用尽。这里使用重定向到标准I/O的方式来避免生成新的fd，首先close()关闭标准I/O的fd，然后使用dup()复制所需的管道fd（会自动复制到序号最小的fd，即关闭的标准I/O），随后对pipe两侧fd进行关闭（此时只会移除描述符，不会关闭实际的file对象）。
2.pipeline关闭： 在完成素数输出后，需要依次退出pipeline上的所有进程。在退出父进程前关闭其标准输入fd，此时read()将读取到eof（值为0），此时同样关闭子进程的标准输入fd，退出进程，这样进程链上的所有进程就可以退出。
####find
1. 在路径中查找特定文件名的文件。基本直接复制 user/ls.c 文件内容。
2. 有几点需要注意的：1.find功能是在目录中匹配文件名，实现思路是递归搜索整个目录树。2.使用open()打开当前fd，用fstat()判断fd的type，如果是文件，则与要找的文件名进行匹配；如果是目录，则循环read()到dirent结构，得到其子文件/目录名，拼接得到当前路径后进入递归调用。3.注意对于子目录中的.和..不要进行递归。
####xarg
1. 没咋用过这个命令，还是先查的它是啥功能。主要作用是读取标准输出中的信息，然后将其作为标准输入，拼接到下一个命令后面来执行。
2. 注意的一点是，利用 fork() 让命令在子进程中执行后，函数并没有结束，还需要继续读取，因此一些变量需要继续设置新的值。
####学习到的关于xv6系统调用的一些知识：
xv6系统调用流程
Lab中对system call的使用很简单，看起来和普通函数调用并没有什么区别，但实际上的调用流程是较为复杂的。我们很容易产生一些疑问：系统调用的整个生命周期具体是什么样的？用户进程和内核进程之间是如何切换上下文的？系统调用的函数名、参数和返回值是如何在用户进程和内核进程之间传递的？

1.用户态调用

在用户空间，所有system call的函数声明写在user.h中，调用后会进入usys.S执行汇编指令：将对应的系统调用号（system call number）置于寄存器a7中，并执行ecall指令进行系统调用，其中函数参数存在a0~a5这6个寄存器中。ecall指令将触发软中断，cpu会暂停对用户程序的执行，转而执行内核的中断处理逻辑，陷入（trap）内核态。

2.上下文切换

中断处理在kernel/trampoline.S中，首先进行上下文的切换，将user进程在寄存器中的数据save到内存中（保护现场），并restore（恢复）kernel的寄存器数据。内核中会维护一个进程数组（最多容纳64个进程），存储每个进程的状态信息，proc结构体定义在proc.h，这也是xv6对PCB（Process Control Block）的实现。用户程序的寄存器数据将被暂时保存到proc->trapframe结构中。

3.内核态执行

完成进程切换后，调用trap.c/usertrap()，接着进入syscall.c/syscall()，在该方法中根据system call number拿到数组中的函数指针，执行系统调用函数。函数参数从用户进程的trapframe结构中获取(a0~a5)，函数执行的结果则存储于trapframe的a0字段中。完成调用后同样需要进程切换，先save内核寄存器到trapframe->kernel_*，再将trapframe中暂存的user进程数据restore到寄存器，重新回到用户空间，cpu从中断处继续执行，从寄存器a0中拿到函数返回值。

至此，系统调用完成，共经历了两次进程上下文切换：用户进程 -> 内核进程 -> 用户进程，同时伴随着两次CPU工作状态的切换：用户态 -> 内核态 -> 用户态。
##实验结果
![](https://img-blog.csdnimg.cn/6b2a86c76cad44b7ab9a9e1c433d73b8.png#pic_center)

ps.在完成实验时每个实验在本地分别另外建立文件夹，将git下来的源文件复制进去；为了方便显示将图片以网络链接的形式粘贴，水印为本人