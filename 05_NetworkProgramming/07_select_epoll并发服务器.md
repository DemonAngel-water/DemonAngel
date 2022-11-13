[Toc]

# 1 IO模型

如果用户想要读写数据，没有数据可读，或者写不进去，但是我们一定要写入或读取成功，怎么实现

1. 阻塞IO

   阻塞的等待，等到数据可读时、可以写时，再操作(等待期间干不了其他事情)

2. 非阻塞IO

   不等待，隔一段时间访问一次，直到达到目的未知(中间隔的时间可以干一些事情)

3. IO多路复用

   让一个进程/线程监听文件，看是否可以操作，等能够操作时，将谁能操作发送给自己，然后再去操作(如果需要访问多个文件，就有必要找一个人专门监听这些文件，然后一有可以操作的文件，就可以直接去操作。)

4. 信号驱动IO(有一定的异步性)SIGIO

   可以让系统在你想要操作的文件可操作时，给你发一个信号，然后再去读写(也不需要阻塞，当数据来时，只需要根据信号执行相应的操作就可以)

5. 异步IO 

   直接告诉被操作文件，需要它的数据，当他把数据获取来时，直接将数据送给这个进程就可以了(这种方式不阻塞，用户进程依然可以做自己的事情，压根儿就不用担心是否有数据可读)

# 2 IO多路复用

让操作系统去监听文件状态，让操作系统，维护一个文件描述符的集合，让其表示这些文件描述符是否可以操作的集合。如果当有文件可以读写时，操作系统维护的状态就会发生改变，需要将这个状态告知给用户进程，此时用户进程才知道谁是可以读写的，此时再去操作文件。

## 2.1 select

select系统调用，select是操作系统给用户提供的IO多路复用的系统调用。

```c
/* 头文件 */
#include <sys/select.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

/* 函数格式 */
int select(int nfds, fd_set *readfds, fd_set *writefds,
		   fd_set *exceptfds, struct timeval *timeout);
/*
 * 功能：
 *      发起让内核监听文件描述符的请求，并获得文件描述符监听状态。
 * 参数：
 *		nfds:文件描述符集合的监听范围
 *		readfds:监听读请求的文件描述符集合的指针
 *		writefds:监听写请求的文件描述符集合的指针
 *		exceptfds:监听异常状态的文件描述符集合的指针
 *		timeout:超时时间的结构体指针
 * 返回值：
 *			成功：返回三个文件描述符集合准备好的文件描述符个数的和
 *			超时：返回0(即在规定时间内，没有任何文件描述符准备好)
 *			失败：返回-1
 * 注意：由于readfds在select之后会被监听后的状态替换，而我们需要反复select时，就需要，
 *       维护一份一直需要监听的文件描述符的列表，而这个文件
 *		 描述符的列表的就是readfds，为了防止其被更改，应该传给select一个readfds的备份。
 */
```

```c
struct timeval
{
    long tv_sec;  /*秒*/
    long tv_usec; /*毫秒*/
};


typedef long int __fd_mask;

typedef struct
{
    __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
} fd_set;

#define FD_SETSIZE __FD_SETSIZE
#define NFDBITS __NFDBITS

#define __NFDBITS (8 * (int)sizeof(__fd_mask))
#define __FD_SETSIZE 1024

__NFDBITS --> 8 * 4 == 32;
fd_set --> long[1024 / 32] --> long[32];
fd_set --> long arr[32];
//fd_set 是32long，在32bit机器上，是32 * 4 * 8 = 1024 bit;

void FD_CLR(int fd, fd_set *set);
/*
 * 从文件描述符集合删除指定的文件描述符
 * 将表示文件描述符的bit置0
 */

int FD_ISSET(int fd, fd_set *set);
/*
 * 判断文件描述符监听后是不是在这个集合里
 * 将表示文件描述符的bit是否为1，如果是1，则说明其准备好了，
 * 表达式的值是非零(真)，如果是0，则说明其没有准备好，表达式的值为0
 */

void FD_SET(int fd, fd_set *set);
/*
 * 将文件描述符加入集合
 * 将表示文件描述符的bit置1
 */

void FD_ZERO(fd_set *set);
/*
 * 清空文件描述符的集合(将1024个bit全部清零)
 */

/* Access macros for `fd_set'.  */
#define FD_SET(fd, fdsetp) __FD_SET(fd, fdsetp)
#define FD_CLR(fd, fdsetp) __FD_CLR(fd, fdsetp)
#define FD_ISSET(fd, fdsetp) __FD_ISSET(fd, fdsetp)
#define FD_ZERO(fdsetp) __FD_ZERO(fdsetp)

#define __FD_ZERO(set)                                                 \
    do                                                                 \
    {                                                                  \
        unsigned int __i;                                              \
        fd_set *__arr = (set);                                         \
        for (__i = 0; __i < sizeof(fd_set) / sizeof(__fd_mask); ++__i) \
            __FDS_BITS(__arr)                                          \
            [__i] = 0;                                                 \
    } while (0)


#define __FD_SET(d, set) \
    ((void)(__FDS_BITS(set)[__FD_ELT(d)] |= __FD_MASK(d)))
#define __FD_CLR(d, set) \
    ((void)(__FDS_BITS(set)[__FD_ELT(d)] &= ~__FD_MASK(d)))
#define __FD_ISSET(d, set) \
    ((__FDS_BITS(set)[__FD_ELT(d)] & __FD_MASK(d)) != 0)

#define __FDS_BITS(set) ((set)->__fds_bits)

/*
 * 让文件描述符/32 （文件描述符的取值范围是0-1023）
 * 结果的范围(0-31)
 */
#define __FD_ELT(d) ((d) / __NFDBITS)

/*
 * 首先让d % 32，余数就是表示文件描述符状态的行中所在的列的位置。
 */
#define __FD_MASK(d) ((__fd_mask)(1UL << ((d) % __NFDBITS)))
```

一个文件描述符的状态只能是准备好或者没有准备好，所有一个bit位就能够代表1个文件描述符的状态。

而fd_set拥有1024 bits，所以fd_set能最大监听1024个文件描述符，即从0 - 1023

### 2.1.1 如何操作fd_set

|  0   |  1   |  2   |  3   |  4   |  5   |  ……  | 1021 | 1022 | 1023 |
| :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: | :--: |

实际上是一个long[32]

|        |   31   |   30   |   29   |   28   |   27   | ...... |   4    |   3    |   2    |   1    |   0    |
| :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: | :----: |
|   31   |   0    |   0    |   0    |   0    |   0    | ...... |   0    |   0    |   0    |   0    |   0    |
|   30   |   0    |   0    |   0    |   0    |   0    | ...... |   0    |   0    |   0    |   0    |   0    |
|   29   |   0    |   0    |   0    |   0    |   0    | ...... |   0    |   0    |   0    |   0    |   0    |
|   28   |   0    |   0    |   0    |   0    |   0    | ...... |   0    |   0    |   0    |   0    |   0    |
| ...... | ...... | ...... | ...... | ...... | ...... | ...... | ...... | ...... | ...... | ...... | ...... |
|   1    |   0    |   0    |   0    |   0    |   0    | ...... |   0    |   0    |   0    |   0    |   0    |
|   0    |   0    |   0    |   0    |   0    |   0    | ...... |   0    |   0    |   0    |   0    |   0    |


### 2.1.2 select搭建并发服务器的流程

```
listenfd = socket()
bind
listen
fd_set readfds, tempfds;
FD_ZERO(&readfds);
FD_SET(listenfd, &readfds);
while (1)
{
    tempfds = readfds;
    ret = select(FD_SETSIZE, &tempfds, NULL, NULL, NULL);
    if (ret > 0)
    {
        int i = 0;
        for (i = 3; i < FD_SETSIZE; i++)
        {
            if (FD_ISSET(i, &tempfds))
            {
                /* 说明监听套接字可读，即有客户端连接服务器 */
                if (i == listenfd)
                {
                    connfd = accpet();
                    if (connfd > 0)
                    {
                        FD_SET(connfd, &readfds);
                    }
                }
                else
                {
                    do_work() 
                    if (recv - return - value == 0)
                    {
                        FD_CLR(i, &readfds);
                        close(i);
                    }
                }
            }
        }
    }
}
close(listenfd);
```

### 2.1.3 select示例


```cpp
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <sys/time.h>
#include <sys/select.h>

int main()
{
	//socket
	int listenfd = socket(AF_INET, SOCK_STREAM, 0);
	if(listenfd < 0)
	{
		puts("socket error.");
		return -1;
	}
	puts("socket success.");
	//bind
	struct sockaddr_in myser;
	myser.sin_family = AF_INET;
	myser.sin_port = htons(8888);
	myser.sin_addr.s_addr = htonl(INADDR_ANY);

	int ret = bind(listenfd, (struct sockaddr *)&myser, sizeof(myser));
	if(ret != 0)
	{
		puts("bind error.");
		close(listenfd);
		return -1;
	}
	puts("bind success.");
	//listen
	ret = listen(listenfd, 5);
	if(ret != 0)
	{
		puts("listen error.");
		close(listenfd);
		return -1;
	}
	puts("listen success.");
	//define readfds and tempfds
	fd_set readfds, tempfds;
	//FD_ZERO readfds
	FD_ZERO(&readfds);
	FD_ZERO(&tempfds);
	//FD_SET listenfd to readfds
	FD_SET(listenfd, &readfds);
	int i = 0;
	//do loop
	while(1)
	{
		//copy readfds to tempfds
		tempfds = readfds;
		//select
		ret = select(FD_SETSIZE, &tempfds, NULL, NULL, NULL);
		if(ret > 0)
		{
			//for loop to FD_ISET fd 
			for(i = 3; i < FD_SETSIZE; i++)
			{
				if(FD_ISSET(i, &tempfds))
				{
					//if listenfd in tempfds(after select)
					if(i == listenfd)
					{
						//accept and FD_SET connfd to readfds
						struct sockaddr_in mycli;
						int len = sizeof(mycli);
						int connfd = accept(i, (struct sockaddr *)&mycli, &len);
						if(connfd > 0)
						{
							puts("accept success.");
							FD_SET(connfd, &readfds);
						}
						else
						{
							continue;
						}
					}
					else
					{
						//else connfd in tempfds (after select) do work
						char buf[100];
						memset(buf, 0, sizeof(buf));
						ret = recv(i, buf, sizeof(buf), 0);
						if(ret > 0)
						{
							puts(buf);
							send(i, buf, sizeof(buf), 0);
						}
						//if recv return value == 0, FD_CLR connfd from readfds
						else if(ret == 0)
						{
							FD_CLR(i, &readfds);
							//close connfd
							close(i);
						}
					}
				}
			}
		}
	}
	//close listenfd
	close(listenfd);
	return 0;
}
```

## 2.2 epoll机制实现并发服务器

### 2.2.1 epoll机制下的并发服务器的搭建流程

由于select和poll每次都需要遍历监听队列有些麻烦，select监听队列的大小比较小，所以引入了使用红黑树进行快速索引和监听队列元素可以更多的epoll来实现并发服务器的搭建。

```cpp
#include <xxx.h>

/* 定义一个epoll监听成员的个数 */
#define MAX_EVENTS 100

int main()
{
    /* 定义epoll使用的结构体和结构体数组 */
    struct epoll_event ev, events[MAX_EVENTS];
	int listen_sock, conn_sock, nfds, epollfd;
	
	socket();
	bind();
	listen();
	
	/* 初始化epoll */
	epollfd = epoll_create(100);
	if(-1 == epollfd)
	{
		perror("epoll_create");
		close(listen_sock);
		return -1;
	}
	
	/* add listen_sock int epollfd */
	ev.events = EPOLLIN;
	ev.data.fd = listen_sock;
	if(-1 == epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev))
	{
		perror("epoll_ctl:listen_sock");
		exit(EXIT_FAILURE);
	}
	
	while(1)
	{
		nfds = epoll_wait(epollfd, enevts, MAX_EVENTS, -1);
		if(-1 == nfds)
		{
			perror("epoll_pwait");
			exit(EXIT_FAILURE);
		}
		int n;
		for(n = 0; n < nfds; ++n)
		{
			if(events[n].data.fd == listen_sock)
			{
				conn_sock = accept(listen_sock, (struct sockaddr *)&myclient, &len);
				/* setnonblocking conn_sock */
				int flags = fcntl(conn_sock, F_GETFL);
				if(flags < 0 || fcntl(conn_sock, F_SETFL, flags | O_NONBLOCK) < 0)
				{
					puts("fcntl conn_sock nonblock error");
					break;
				}
				
				ev.events = EPOLLIN | EPOLLET;
				ev.data.fd = conn_sock;
				if(-1 == epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock, &ev))
				{
					perror("epoll_ctl: conn_sock");
					exit(EXIT_FAILURE);
				}
				puts("accept client sucess");
			}
			else
			{
				/* recv and send */
				char buf[30];
				memset(buf, 0, sizeof(buf));
				int rl = recv(events[n].data.fd, buf, sizeof(buf), 0);
				if(rl > 0)
				{
					puts(buf);
					send(events[n].data.fd, buf, strlen(buf), 0);
				}
				else if(0 == rl)
				{
					/* close */
					close(events[n].data.fd);
					epoll_ctl(epollfd, EPOLL_CTL_DEL, events[n].data.fd, &ev);
				}
			}
		}
	}
	/* close */
	close(listen_sock);
	return 0;
}
```

#### 2.2.1.1 epoll_create

```c
/* 需要的头文件 */
#include <sys/epoll.h>

/* 函数原型 */
int epoll_create(int size);
/*
 * 功能描述：
 * 		该函数创建一个epoll的实例，自Linux2.6.8内核之后，size就被忽略了，但是必须大于0，
 * 返回值：
 * 		该函数的返回是一个文件描述符，这个文件描述符用来说明一个新的epoll实例，
 *		这个文件描述符被epoll_ctl和epoll_wait所使用。当该文件描述符不在被使用时，
 * 		需要close去关闭。
 * 		成功返回文件描述符，失败返回-1
 */
```

#### 2.2.1.2 epoll_ctl

```c
	/* 需要的头文件 */
#include <sys/epoll.h>

/* 函数原型 */
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
/*
 * 功能描述：
 * 		该函数是控制epoll文件描述符的接口，通常用于添加，修改或者从epoll
 * 		监听的对象中移出一个文件描述符
 * 参数：
 * 		epfd: 是epoll的操作对象，是epoll_create函数的返回值
 *		op的取值对象有以下一些：
 * 			EPOLL_CTL_ADD: 添加一个fd到epfd中
 * 			EPOLL_CTL_MOD: 修改已经注册的fd的监听事件
 * 			EPOLL_CTL_DEL: 删除一个fd从epfd中
 * 		event: 表示被监听的文件描述符被监听的事件
			typedef union epoll_data {
               void        *ptr;
               int          fd;
               uint32_t     u32;
               uint64_t     u64;
            } epoll_data_t;

           struct epoll_event {
               uint32_t     events;      // Epoll events 
               epoll_data_t data;        // User data variable
           };
 *		events成员的取值：
			EPOLLIN				表示对应被监听的文件描述符可读
			EPOLLOUT			表示对应被监听的文件描述符可写
			EPOLLPRI			表示对应的文件描述符有紧急的数据可读
			EPOLLERR			表示对应的文件描述符发生了错误
			EPOLLHUP			表示对应的文件描述符被挂断
			EPOLLET				将EPOLL设置为边缘触发（Edge Triggered）模式，这是相对于水平触发（Level Triggered）来说的
			EPOLLONESHOT		只监听一次事件，当监听完这次事件后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列中
 */
```

#### 2.2.1.3 epoll_wait

```c
/* 需要的头文件 */
#include <sys/epoll.h>

/* 函数原型 */
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
/*
 * 功能描述：
 * 		等待事件的产生，类似于select()调用
 * 参数：
 *		events: 用来从内核得到事件的集合
 * 		maxevents: 表示每次能处理的最大事件数，告知内核这个events有多大，maxevents的值不呢个大于创建epoll_create()时的size
 * 		timeout: 超时时间（微妙，0会立即返回，-1将不确定，也有说法是永久阻塞）。
 * 返回值：
 * 		返回需要处理的事件数目，如返回0表示已经超时 
 */
```

### 2.2.2 epoll示例


```cpp
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <fcntl.h>
#include <unistd.h>

// define epoll listen number
#define MAX_EVENTS 100

int main()
{
    // socket
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd < 0)
    {
        puts("socker error.");
        return -1;
    }
    puts("socket success.");

    // bind
    struct sockaddr_in myser;
    memset(&myser, 0, sizeof(myser));
    myser.sin_family = AF_INET;
    myser.sin_port = htons(8888);
    myser.sin_addr.s_addr = htonl(INADDR_ANY);

    int ret = bind(listenfd, (struct sockaddr *)&myser, sizeof(myser));
    if(ret != 0)
    {
        puts("bind error.");
        close(listenfd);
        return -1;
    }
    puts("bind success.");
    // listen
    ret = listen(listenfd, 5);
    if(ret != 0)
    {
        puts("listen error.");
        close(listenfd);
        return -1;
    }
    puts("listen success.");
    // init epoll
    struct epoll_event ev, events[MAX_EVENTS];
    int nfds;
    int epollfd = epoll_create(MAX_EVENTS);
    if(epollfd < 0)
    {
        puts("epoll create error.");
        close(epollfd);
        return -1;
    }
    puts("epoll create success.");
    // add listen into epolled
    // ev.events = EPOLLIN|EPOLLET; 		// 边缘触发
    ev.events = EPOLLIN;					// 水平触发
    ev.data.fd = listenfd;

    if(-1 == epoll_ctl(epollfd, EPOLL_CTL_ADD, listenfd, &ev))
    {
        puts("epoll_ctl: listenfd error.");
        close(listenfd);
        return -1; 
    }
    struct sockaddr_in myclient;
    int len = sizeof(myclient);
    int connfd;
    while(1)
    {
        // epoll_wait
        nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
        if(-1 == nfds)
        {
            puts("epoll_wait error.");
            return -1;
        }
        // forloop check listen sucess fd
        int n = 0;
        for(n = 0; n < nfds; n++)
        {
            // if events.fd == listenfd accept client
            if(listenfd == events[n].data.fd)
            {
                // accept
                connfd = accept(events[n].data.fd, (struct sockaddr *)&myclient, &len);
                if(connfd < 0)
                {
                    puts("accept error.");
                    break;
                }
                // set connfd O_NONBLOCK
                int flags = fcntl(connfd, F_GETFL);
                if(flags < 0 || fcntl(connfd, F_SETFL, flags | O_NONBLOCK) < 0)
                {
                    puts("fcntl connfd nonblock error.");
                    break;
                }
                // add connfd to epollfd
                ev.events = EPOLLIN | EPOLLET;
                ev.data.fd = connfd;
               
                if(-1 == epoll_ctl(epollfd, EPOLL_CTL_ADD, connfd, &ev))
                {
                    puts("epoll_ctl: connfd error.");
                    return -1;
                }
                puts("accept client success.");
            }
            else
            {
                // else send and recv
                char buf[30];
                memset(buf, 0, sizeof(buf));
                int rl = recv(events[n].data.fd, buf, sizeof(buf), 0);
                if(rl > 0)
                {
                    puts(buf);
                    send(events[n].data.fd, buf, strlen(buf), 0);
                }
                else if(rl == 0)
                {
                    // if recv ret == 0, close eventfd
                    close(events[n].data.fd);
                    // remove eventfd from epollfd
                    epoll_ctl(epollfd, EPOLL_CTL_DEL, events[n].data.fd, &ev);
                }
            }
        }
    }
    // close listenfd
    close(listenfd);
    return 0;
}
```
