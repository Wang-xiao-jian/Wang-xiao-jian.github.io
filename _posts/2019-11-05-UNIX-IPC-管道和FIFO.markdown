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

> PIPE and FIFO...

管道是最初Unix IPC的形式，但是它最根本的局限是没有名字，因此管道只能应用于有亲缘关系的进程之间。它的数据是维护在内核中的，因此每一次对数据的访问都是对内核的系统调用。对管道中数据的访问通常是使用read和write方法。

##### 管道的创建

所有式样的Unix都提供管道，它由`pipe`函数创建：

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

之前提到，pipe能够创建一个单向的数据流，那么我们可以在父进程中创建两个这样的数据流pipe1[2]和pipe2[2]，通过`fork`生成子进程(通过fork产生的子进程会获得父进程中所有打开着的描述符的副本)。然后我们在父进程中关闭pipe1的读描述符，在子进程中关闭pipe1的写描述符，这样就创建了一个父进程到子进程的单向数据流；同理在父进程中关闭pipe2的写描述符，在子进程中关闭pipe2的读描述符，这样就创建了一个子进程到父进程的单向数据流。就此，父子进程间的双向通信已经建立。

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

FIFO由`mkfifo`方法创建，在创建之后还需要通过`open`或者`fopen`(某个I/O打开函数)打开：

```c
#include <sys/stat.h>	//mkfifo, 6个权限常值

#include <sys/types.h>

int mkfifo(const char* pathname, mode_t mode);
//若成功则返回0，若出错则返回-1
```

pathname就是一个普通的Unix路径名，它就是该FIFO的名字。参数mode用来指定文件的权限位，当然它会被进程的文件创建掩码所修改。

此外`mkfifo`暗含了`O_CREAT | O_EXCL`，这表明了要么创建一个新的FIFO，要么如果该FIFO已存在，就会返回一个`EEXIST`错误。因此当创建一个FIFO的时候，检查是否成功很必要，如果返回`EEXSIT`错误就应该调用open来打开它。

当使用mkfifo方法成功创建了一个FIFO之后，如果想要使用它传递消息，那么还必须使用open来打开它：

```c
#include <sys/types.h>

#include <sys/stat.h>

#include <fcntl.h>	//O_RDONLY, O_WRONLY

int open(const char *path, int oflag, ... );
```

FIFO可以打开来读(指定O_RDONLY)或者打开来写(指定O_WRONLY)，但是FIFO不允许打开来同时读写。对FIFO的write总是往末尾添加数据，对其的read总是从头开始读取数据，因为first in, first out，因此也不能对FIFO调用`lseek`，如果你想插队它就会给你返回一个`ESPIPE`错误。

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
		printf("cant create %s\n", FIFO1);
	if ( (mkfifo(FIFO2, FILE_MODE) < 0) && errno != EEXIST) {
		unlink(FIFO1);
		printf("cant create %s\n", FIFO2);
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

其实在这里有个很微妙的问题：如果我们交换父进程或者子进程的open的顺序，程序就会不工作。这是因为当还没有一个进程打开FIFO来写的话，那么打开这个FIFO来读的进程就会被阻塞，直到有进程打开这个FIFO来写。如果将父进程的open交换顺序的话，父子进程将打开相同的FIFO来读，但是当时并没有进程打开FIFO来写，由此造成*死锁*(deadlock)。这其实是接下来要写的***管道和FIFO的额外属性***的一部分。

##### 管道和FIFO的额外属性

我本来接下来想先写FIFO的真正表现其优势的应用部分，但是发现这部分中有一部分还是要先了解接下来要讲的FIFO的额外属性有关系，所以就先写这部分我觉得光看书并不容易理解的知识吧。

这些是我们需要知道的就管道和FIFO打开，读出，写入这三方面更为详细的相关属性。

###### 非阻塞

一个描述符能够以两种方式设置为非阻塞:

- 在调用open时，直接指定`O_NONBLOCK`标志，就比如***FIFO的简单应用***中,第28行代码就可以写为：

  ```
  readfd = open(FIFO1, O_RDONLY | O_NONBLOCK, 0);
  ```

- 但是之前说到的管道并不会调用open来打开，那么就只能使用第二种方法：对于一个已经打开的描述符，可以调用`fcntl`来启用O_NONBLOCK。

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

- 以只读/只写open一个FIFO的时候，如果之前没有对该FIFO以只写/只读调用open，那么该open调用就会阻塞到直到以相应的模式open FIFO。这在***FIFO的简单运用***中有提到过。但是如果指定了非阻塞标志的话，以只读模式打开一个FIFO都会直接成功返回；而如果以只写模式打开一个FIFO并且之前没有以只读打开该FIFO，那么就会返回`ENXIO`错误。
- 尝试从空管道或FIFO read的时候，如果该FIFO在其他进程以写打开了，那么就会阻塞到以只写模式打开该FIFO的进程写入数据，或者该进程关闭以只写打开的该FIFO。如果指定了非阻塞标志的话，就会返回一个`EAGAIN`错误，它的意思其实是资源暂时不可用，就是要你再次尝试的意思。在缓冲区满但是却要以非阻塞模式往里写入数据或者缓冲区为空却要以非阻塞模式往里读出数据的情境下就会出现这个错误。在当前的情境下，我自己按白话理解就是：“现在店里没货，但是有人去进货了，啥时候回来不知道，你待会再来吧”。
- 如果read请求读出的数据量大于可用的数据量时，那么只返回这些可用的数据，所以我们需要处理读取到的数据量小于请求的数据量的情况。
- 尝试从空管道或FIFO read的时候，如果该FIFO并没有以只写被其他进程打开，那么read返回0，表明没有数据可读。
- 如果往一个没有以读打开的空管道或FIFO写入数据的话，会产生一个`SIGPIPE`信号，该信号的默认行为是终止进程，如果忽略了该信号或者捕获了信号做了处理，那么write就会返回`EPIPE`以表明这个错误。
- 关于write

  - 当要写入的数据量小于`PIPE_BUF`(一个Posix的限定值)时，那么write的操作可以保证是原子性的。也就是说在差不多时候，两个进程都往同一个FIFO中写入数据，要么先全部写入进程1的数据，再全部写入进程2的数据，或者反过来，总之不会混杂这两个进程的数据。但是如果写入的数据量大于PIPE_BUF时，则write不能保证原子性。

  - 是否指定非阻塞标志并不会影响write的原子性，影响write原子性的只能是写入数据量的大小。

  - 如果指定了非阻塞标志并且写入的数据量小于PIPE_BUF，那么：

    - 如果管道或FIFO中剩余的空间能全部容纳，那么就成功写入所有的数据。

    - 如果剩余空间不能全部容纳，但是又不希望进程投入睡眠以等待足够的空间，那么write就会立即返回EAGAIN错误，意思就是“客官现在满座了，您先出去转转，待会再来”。虽然它不提供取号叫号的服务，但也因此不会有类似“今天的号已经取完了”这样令人心碎的事情发生不是么？

  - 如果指定了非阻塞标志并且接入的数据量大于PIPE_BUF，那么：

    - 如果管道或FIFO有至少1字节的空间，那么就写入管道或FIFO能容量的数目，同时这个数值作为write的返回值。此时write的操作就不是原子性了。
    - 如果管道或FIFO满了，那么立即返回EAGAIN错误。

##### FIFO高级应用——单个服务器，多个客户

FIFO因为有名字，所以可以用于没有亲缘关系的进程之间的IPC手段。一种比较合理的场景是：一个服务器程序以只读打开一个众所周知的FIFO，然后等待某个时刻客户以只写打开该FIFO并发送请求。

这样很容易实现从客户到服务器的单向通信。但是反过来，想要实现从服务器到客户的单向数据流就不容易了，服务器如何知晓每个进程打开的FIFO名呢？我们可以事先约定客户打开的FIFO名字的前缀，然后加上客户的进程号作为客户打开的FIFO的名字的后缀，然后客户在向服务器发送请求的时候以`pid+' '+请求`的形式发送给服务器。

在设计服务器的代码时，我用的是并发服务器的模型，针对每个客户的请求都生成一个子进程来处理。这是考虑万一客户发送了请求之后，却迟迟不打开属于它的FIFO来读，这样服务器就会阻塞在以只写模式打开客户的FIFO这个步骤上，从而无法服务其它的客户，这其实就是*拒绝服务型攻击*。就算不考虑拒绝服务型攻击，迭代服务器也会让客户花费额外的时间去等待服务器完成其它客户请求。

当然，并发模式也可以被攻击：一个恶意的客户可以fork多个进程去连接服务器但是拒绝服务，从而导致服务器fork的进程数达到上限。我们可以在可能发生阻塞的地方设置超时来规避这种情况。

如果从一个空管道或FIFO read，并且此时没有其他进程以只写打开这个FIFO的话，那么read就返回0，表明读取结束。作为服务器来说，如果不做特殊处理，每当一个客户终止，那么服务器以只读打开的众所周知的FIFO就会为空，服务器的read就会返回0，我们将不得不关闭该FIFO并且调用open以只读再次打开，只不过这次调用会阻塞到下个客户的到来（试想如果我们不关闭该FIFO，而是不断调用read尝试去从该空FIFO中读取会发生什么？每次read都会立即返回0，直到下一个客户到来以只写打开该FIFO，然而我们不知道何时客户才会到来，所以不断调用read轮询无疑是在浪费CPU时间）。

特殊处理就是：在服务器以只读打开该众所周知的FIFO之后，再次调用open以只写打开该FIFO，返回的writefd我们从不使用，也从不关闭，这样当一个客户终止的时候，服务器只是阻塞在read的调用上，我们不需要关闭该FIFO再重新打开，也不需要不断调用read轮询导致CPU时间的浪费。简化了我们的代码也减少了open的调用。

```c++
//服务器代码

#include <sys/stat.h>

#include <sys/types.h>

#include <errno.h>

#include <stdio.h>

#include <unistd.h>

#include <fcntl.h>

#include <stdlib.h>

#include <string.h>

#include <sys/wait.h>

#define MAX_BUF 1024

#define SERV_FIFO "/tmp/serv.fifo"

#define FILE_MODE (S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH)

int main(int argc, char const *argv[])
{
	if (mkfifo(SERV_FIFO, FILE_MODE) < 0 && errno != EEXIST) {
		printf("cant create fifo : %s\n", SERV_FIFO);
		_exit(0);
	}

	int readfd = open(SERV_FIFO, O_RDONLY);
	int yummyfd = open(SERV_FIFO, O_WRONLY);	//这样没有客户端进程以只写打开serv_fifo时也不用关闭readfd重新打开

	int n;
	char buff[MAX_BUF + 1] = { 0 };
	int child_pid;
	while ( (n = read(readfd, buff, MAX_BUF)) > 0) {
		if (buff[n - 1] == '\n') 
			--n;
		buff[n] = '\0';
		
		if ( (child_pid = fork()) == 0) {
			close(yummyfd);
			close(readfd);
			char* ptr;
			if ( (ptr = strchr(buff, ' ')) == NULL) {
				printf("bad request : %s\n", buff);
				_exit(0);
			}
			*ptr++ = 0;	//buff = pid, ptr = pathname

			char fifoname[32] = { 0 };
			snprintf(fifoname, sizeof(fifoname), "/tmp/fifo.%ld", atol(buff));
			int wd;
			if ( (wd = open(fifoname, O_WRONLY)) < 0) {
				printf("cant open fifo : %s\n", fifoname);
				_exit(0);
			}

			int fd;
			if ( (fd = open(ptr, O_RDONLY)) < 0) {
				printf("cant open %s : %s\n", ptr, strerror(errno));
				snprintf(buff + n, sizeof(buff) - n, "cant open : %s", strerror(errno));
				n = strlen(ptr);
				write(wd, ptr, n);
				close(wd);
			} else {
				while ( (n = read(fd, buff, MAX_BUF)) > 0) {
					write(wd, buff, n);
				}
				close(wd);
				close(fd);
			}
			_exit(0);
		}
		
		waitpid(child_pid, NULL, 0);
	}

	return 0;
}
```

```c++
//客户代码

#include <unistd.h>

#include <sys/types.h>

#include <sys/stat.h>

#include <errno.h>

#include <string.h>

#include <stdio.h>

#include <fcntl.h>

#define MAX_BUF 1024

#define SERV_FIFO "/tmp/serv.fifo"

#define FILE_MODE (S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH)

int main(int argc, char const *argv[])
{
	char buff[MAX_BUF + 1] = { 0 };
	char fifoname[32] = { 0 };
	pid_t pid = getpid();
	snprintf(fifoname, sizeof(fifoname), "/tmp/fifo.%d", pid);
	if (mkfifo(fifoname, FILE_MODE) < 0 && errno != EEXIST) {
		printf("cant create fifo %s : %s\n", fifoname, strerror(errno));
		_exit(0);
	}
	snprintf(buff, sizeof(buff), "%d ", pid);
	int len = strlen(buff);


	int n;
	if ( (n = read(STDIN_FILENO, buff + len, MAX_BUF - len)) <= 0) {
		printf("bad input\n");
		_exit(0);
	}


	len = strlen(buff);

	int wd;
	if ( (wd = open(SERV_FIFO, O_WRONLY)) < 0) {
		printf("cant open %s : %s\n", SERV_FIFO, strerror(errno));
		_exit(0);
	}

	write(wd, buff, len);

	int rd = open(fifoname, O_RDONLY);

	memset(buff, 0, sizeof(buff));

	while ( (n = read(rd, buff, MAX_BUF)) > 0) {
		write(STDOUT_FILENO, buff, n);
	}

	return 0;
}
```

##### 管道和FIFO的一个特征

如果仔细思考一下不难发现管道和FIFO是以数据流的形式传递数据的，类似于TCP和UDP。它不存在记录边界，就比如说一个管道里有100字节的数据，应用程序无法判断这是单次写入100字节，还是分100次每次写一个字节或者其他某种总共100字节的写操作的组合。一个进程写了60字节，另一个字节写了40字节，这也是可能的。如果想要将这种字节流分隔成单个记录，那么是需要应用程序来实现的。

一种方法是定义一个结构体，该结构体包含一个`long`类型的数据表示消息的长度，然后另一个成员是`char数组`来储存消息内容。这样应用程序读取的时候先读取4个字节获得消息的长度，然后再调用read读取单个记录消息。

##### 管道和FIFO的限制

- OPEN_MAX	一个进程能够打开的最大描述符数量(Posix要求至少为16)
- PIPE_BUF       可原子地往一个管道或FIFO写入的最大数据量(Posix要求至少为512)

