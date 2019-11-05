---
layout: post
title: "UNIX IPC 管道和FIFO"
data: 2019-11-05 11:06:00 +0800
author: "riki"
header-img: "img/20191105.jpg"
tags:
- UNIX
- IPC
- 管道
typora-root-url: ..
---

> 未完待续

#### 管道

管道是最初Unix IPC的形式，但是它最根本局限是没有名字，因此管道只能应用于有亲缘关系的进程之间。它的数据是维护在内核中的，因此每一次对数据的访问都是对内核的系统调用。对管道中数据的访问通常是使用read和write方法。

##### 管道的创建

所有式样的Unix都提供管道，它由pipe函数创建：

```c
#include <unistd.h>

int pipe(int fd[2]);
//若成功返回0，出错返回-1
```

它返回了两个描述符：fd[1]用来写，fd[0]用来读。由此它创建了一个单向的数据流，由fd[1]流向fd[0]。

##### 管道的关闭

调用pipe就相当于创建并打开了返回的两个读写描述符，关闭则使用close。

##### 管道的应用

虽然单个进程就能创建管道，但是我们很少会在单个进程内使用，因为没什么意义。管道经典的应用是为父子进程提供通信的手段。

之前提到，pipe能够创建一个单向的数据流，那么我们可以在父进程中创建两个这样的数据流pipe1[2]和pipe2[2]，通过fork生成子进程(通过fork产生的子进程会获得父进程中所有打开着的描述符的副本)。然后我们在父进程中关闭pipe1的读描述符，在子进程中关闭pipe1的写描述符，这样就创建了一个父进程到子进程的单向数据流；同理在父进程中关闭pipe2的写描述符，在子进程中关闭pipe2的读描述符，这样就创建了一个子进程到父进程的单向数据流。就此，父子进程间的双向通信已经建立。

以下例子为子进程作为服务器，父进程作为客户。父进程从标准输入获取文件路径并将其发送给服务器，服务器尝试打开该文件，若打开文件失败则返回失败信息，若打开成功则返回文件内容。

```c
#include <unistd.h>

#include <stdio.h>

#include <errno.h>

#include <sys/types.h>

#include <sys/wait.h>

#include <fcntl.h>

#include <string.h>

#define MAX_BUF 1024

void server(int, int);
void client(int, int);

int main()
{
	int pipe1[2], pipe2[2];
	pipe(pipe1);
	pipe(pipe2);	
	pid_t child_pid;
	if (0 == (child_pid = fork())) {
		close(pipe1[0]);
		close(pipe2[1]);

		server(pipe2[0], pipe1[1]);
		_exit(0);
	}

	close(pipe1[1]);
	close(pipe2[0]);
	client(pipe1[0], pipe2[1]);

	waitpid(child_pid, NULL, 0);

	return 0;
}

void server(int rd, int wd)
{
	char buff[MAX_BUF + 1] = { 0 };
	int n;
	
	if ( (n = read(rd, buff, MAX_BUF)) == 0) {
		printf("EOF while reading pathname\n");
		_exit(0);
	}

	buff[n-1] = '\0';
	int fd;

	if ( (fd = open(buff, O_RDONLY)) < 0) {
		snprintf(buff + n, sizeof(buff) - n, ": open error, %s\n", strerror(errno));
		write(wd, buff, strlen(buff));
		_exit(0);
	} else {
		memset(buff, 0, sizeof(buff));
		while ( (n = read(fd, buff, MAX_BUF)) > 0) {
			write(wd, buff, strlen(buff));
			memset(buff, 0, sizeof(buff));
		}
	}
	close(fd);
}

void client(int rd, int wd)
{
	char buff[MAX_BUF + 1] = { 0 };
	fgets(buff, MAX_BUF, stdin);

	write(wd, buff, strlen(buff));
	int n;
	memset(buff, 0, sizeof(buff));
	while ( (n = read(rd, buff, MAX_BUF)) > 0) {
		write(STDOUT_FILENO, buff, n);
		memset(buff, 0, sizeof(buff));
	}
}
```

#### FIFO

FIFO(first in first out)代指先进先出，在Unix中类似于管道，它也是一个单向的数据流，但是与管道不同的是它拥有名字。每一FIFO都有一个路径名与之对应，从而允许无亲缘关系的进程访问同一个FIFO，故而FIFO也称为有名管道。

##### FIFO的创建与打开

FIFO由mkfifo方法创建，在创建之后还需要通过open或者fopen(某个I/O打开函数)打开：

```c
#include <sys/stat.h>	//mkfifo, 6个权限常值

#include <sys/types.h>

int mkfifo(const char* pathname, mode_t mode);
//若成功则返回0，若出错则返回-1
```

pathname就是一个普通的Unix路径名，它就是该FIFO的名字。参数mode用来指定文件的权限位，当然它会被进程等文件创建掩码所修改。

此外mkfifo暗含了O_CREAT | O_EXCL，这表明了要么创建一个新的FIFO，要么如果该FIFO已存在，就会返回一个EEXIST错误。因此当创建一个FIFO的时候，检查是否成功很必要，如果返回EEXSIT错误就应该调用open来打开它。

当使用mkfifo方法成功创建了一个FIFO之后，如果想要使用它传递消息，那么还必须使用open来打开它：

```c
#include <sys/types.h>

#include <sys/stat.h>

#include <fcntl.h>	//O_RDONLY, O_WRONLY

int open(const char *path, int oflag, ... );
```

FIFO可以打开来读(指定O_RDONLY)或者打开来写(指定O_WRONLY)，但是FIFO不允许打开来同时读写。对FIFO的write总是往末尾添加数据，对其的read总是从头开始读取数据，因为first in, first out，因此也不能对FIFO调用lseek，如果你想插队它就会给你返回一个ESPIPE错误。

##### FIFO的简单运用

就***管道的应用***中的代码完全可以用FIFO替代，只需要修改main函数部分的代码。

```c
#include <cstdio>

#include <unistd.h>

#include <sys/types.h>

#include <sys/stat.h>

#include <fcntl.h>

#include <errno.h>

#include <cstring>

#include <sys/wait.h>

#define FIFO1 "/tmp/fifo.1"

#define FIFO2 "/tmp/fifo.2"

#define FILE_MODE (S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH)

const static int MAXLINE = 1024;

int main()
{
	pid_t child_pid;
	int readfd, writefd;

	if ( (mkfifo(FIFO1, FILE_MODE) < 0) && errno != EEXIST)
		printf("camt create %s\n", FIFO1);
	if ( (mkfifo(FIFO2, FILE_MODE) < 0) && errno != EEXIST) {
		unlink(FIFO1);
		printf("camt create %s\n", FIFO2);
	}

	if ((child_pid = fork()) == 0) {
		readfd = open(FIFO1, O_RDONLY, 0);
		writefd = open(FIFO2, O_WRONLY, 0);

		server(readfd, writefd);
		_exit(0);
	}
	writefd = open(FIFO1, O_WRONLY, 0);
	readfd = open(FIFO2, O_RDONLY, 0);
	client(readfd, writefd);
	waitpid(child_pid, NULL, 0);
	unlink(FIFO1);
	unlink(FIFO2);
	_exit(0);
}
```

其实在这里有个很微妙的问题：如果我们交换父进程或者子进程的open的顺序，程序就会不工作。这是因为当还没有一个进程打开FIFO来写的话，那么打开这个FIFO来读的进程就会被阻塞，知道有进程打开这个FIFO来写。如果将父进程的open交换顺序的话，父子进程将打开相同的FIFO来读，但是当时并没有进程打开FIFO来写，由此造成*死锁*(deadlock)。这其实是接下来要写的***管道和FIFO的额外属性***的一部分。

##### 管道和FIFO的额外属性

我本来接下来想先写FIFO的真正表现其优势的应用部分，但是发现这部分中有一部分还是要先了解接下来要讲的FIFO的额外属性有关系，所以就先写这部分我觉得光看书并不容易理解的知识吧。

这些是我们需要知道的就管道和FIFO打开，读出，写入更为详细的相关属性。

###### 非阻塞

一个描述符能够以两种方式设置为非阻塞:

- 在调用open时，直接指定O_NONBLOCK标志，就比如***FIFO的简单应用***中,第28行代码就可以写为：

  ```
  readfd = open(FIFO1, O_RDONLY | O_NONBLOCK, 0);
  ```

- 但是之前说到的管道并不会调用open来打开，那么就只能使用第二种方法：对于一个已经打开的描述符，可以调用fcntl来启用O_NONBLOCK。

  ```c
  #include <fcntl.h>
  
  int flag = fcntl(fd, F_GETFL, 0);	//获得当前文件状态标志
  flag |= O_NONBLOCK;					//启用非阻塞
  fcntl(fd, FSETFL, flag);
  ```


如果直接使用以下一行代码来设置文件状态标志的话，会清除原来的文件状态标志，是不可取的。

​		`fcntl(fd, FSETFL, O_NONBLOCK);`


以下图片中的内容展示了指定非阻塞标志对于打开FIFO只读，打开FIFO只写，从空管道或者空FIFO中读取以及往管道或FIFO中写入的影响：

![](/img/in-post/额外属性.png)

我的理解：

- 以只读/只写open一个FIFO的时候，如果之前没有对该FIFO以只写/只读调用open，那么该open调用就会阻塞到直到以相应的模式open FIFO。这在***FIFO的简单运用***中有提到过。但是如果指定了非阻塞标志的话，以只读模式打开一个FIFO都会直接成功返回；而如果以只写模式打开一个FIFO并且之前没有以只读打开该FIFO，那么就会返回ENXIO错误。

- 尝试从空管道或FIFO read的时候，如果该FIFO在其他进程以写打开了，那么就会阻塞到以只写模式打开该FIFO的进程写入数据，或者该进程关闭以只写打开的该FIFO。如果指定了非阻塞标志的话，就会返回一个EAGAIN错误，它的意思其实是资源暂时不可用，就是要你再次尝试的意思。在缓冲区满但是却要以非阻塞模式往里写入数据或者缓冲区为空却要以非阻塞模式往里读出数据的情境下就会出现这个错误。在当前的情境下，我自己按白话理解就是：“现在店里没货，但是有人去进货了，啥时候回来不知道，你待会再来吧”。

- 如果read请求读出的数据量大于可用的数据量时，那么只返回这些可用的数据，所以我们需要处理读取到的数据量小于请求的数据量的情况。

- 尝试从空管道或FIFO read的时候，如果该FIFO并没有以只写被其他进程打开，那么read返回0，表明没有数据可读。

- 如果往一个没有以读打开的空管道或FIFO写入数据的话，会产生一个SIGPIPE信号，该信号的默认行为是终止进程，如果忽略了该信号或者捕获了信号做了处理，那么write就会返回EPIPE以表明这个错误。

- 关于wirte

  - 当要写入的数据量小于PIPE_BUF(一个Posix的限定值)时，那么write的操作可以保证是原子性的。也就是说在差不多时候，两个进程都往同一个FIFO中写入数据，要么先全部写入进程1的数据，再全部写入进程2的数据，或者反过来，总之不会混在这两个进程的数据。但是如果写入的数据量大于PIPE_BUF时，则write不能保证原子性。

  - 是否指定非阻塞标志并不会影响write的原子性，影响write原子性的只能是写入数据量的大小。

  - 如果指定了非阻塞标志并且写入的数据量小于PIPE_BUF，那么：

    - 如果管道或FIFO中剩余的空间能全部容纳，那么就成功写入所有的数据。

    - 如果剩余空间不能全部容纳，但是又不希望进程投入睡眠以等待足够的空间，那么write就会立即返回EAGAIN错误，意思就是“客官现在满座了，您先出去转转，待会再来”。虽然它不提供取号叫号的服务，但也因此不会有类似“今天的号已经取完了”这样令人心碎的事情发生不是么？

  - 如果指定了非阻塞标志并且接入的数据量大于PIPE_BUF，那么：

    - 如果管道或FIFO有至少1字节的空间，那么就写入管道或FIFO能容量的数目，同时这个数值作为write的返回值。此时write的操作就不是原子性了。
    - 如果管道或FIFO满了，那么立即返回EAGAIN错误。