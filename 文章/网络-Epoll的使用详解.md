# 一. Epoll简介  

EPOLL 的API用来执行类似poll()的任务。能够用于检测在多个文件描述符中任何IO可用的情况。Epoll API可以用于边缘触发(edge-triggered)和水平触发(level-triggered), 同时epoll可以检测更多的文件描述符。以下的系统调用函数提供了创建和管理epoll实例：  

* epoll_create() 可以创建一个epoll实例并返回相应的文件描述符(epoll_create1() 扩展了epoll_create() 的功能)。
* 注册相关的文件描述符使用epoll_ctl()
* epoll_wait() 可以用于等待IO事件。如果当前没有可用的事件，这个函数会阻塞调用线程。  


**边缘触发(edge-triggered 简称ET)和水平触发(level-triggered 简称LT)：**  

epoll的事件派发接口可以运行在两种模式下：边缘触发(edge-triggered)和水平触发(level-triggered)，两种模式的区别请看下面,我们先假设下面的情况：  

1. 一个代表管道读取的文件描述符已经注册到epoll实例上了。
2. 在管道的写入端写入了2kb的数据。
3. epoll_wait 返回一个可用的rfd文件描述符。
4. 从管道读取了1kb的数据。
5. 调用epoll_wait 完成。  

如果rfd被设置了ET，在调用完第五步的epool_wait 后会被挂起，尽管在缓冲区还有可以读取的数据，同时另外一段的管道还在等待发送完毕的反馈。这是因为ET模式下只有文件描述符发生改变的时候，才会派发事件。所以第五步操作，可能会去等待已经存在缓冲区的数据。在上面的例子中，一个事件在第二步被创建，再第三步中被消耗，由于第四步中没有读取完缓冲区，第五步中的epoll_wait可能会一直被阻塞下去。  

下面情况下推荐使用ET模式:  

1. 使用非阻塞的IO。
2. epoll_wait() 只需要在read或者write返回的时候。  

相比之下，当我们使用LT的时候（默认）,epoll会比poll更简单更快速，而且我们可以使用在任何一个地方。  

# 二. API介绍  
先简单的看下EPOLL的API  

## 2.1 创建epoll  

```c++
#include <sys/epoll.h>

int epoll_create(int size);
int epoll_create1(int flags);
```

epoll_create() 可以创建一个epoll实例。在linux 内核版本大于2.6.8 后，这个`size` 参数就被弃用了，但是传入的值必须大于0。  

```console
在 epoll_create () 的最初实现版本时， size参数的作用是创建epoll实例时候告诉内核需要使用多少个文件描述符。内核会使用 size 的大小去申请对应的内存(如果在使用的时候超过了给定的size， 内核会申请更多的空间)。现在，这个size参数不再使用了（内核会动态的申请需要的内存）。但要注意的是，这个size必须要大于0，为了兼容旧版的linux 内核的代码。
```

epoll_create() 会返回新的epoll对象的文件描述符。这个文件描述符用于后续的epoll操作。如果不需要使用这个描述符，请使用close关闭。  

epoll_create1() 如果`flags`的值是0，epoll_create1()等同于epoll_create()除了过时的size被遗弃了。当然`flasg`可以使用 EPOLL_CLOEXEC，请查看 open() 中的O_CLOEXEC来查看 EPOLL_CLOEXEC有什么用。  

返回值: 如果执行成功，返回一个非负数(实际为文件描述符), 如果执行失败，会返回-1，具体原因请查看error.  

## 2.2 设置epoll事件    

```c++
#include <sys/epoll.h>

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

这个系统调用能够控制给定的文件描述符epfd指向的epoll实例，op是添加事件的类型，fd是目标文件描述符。  

有效的op值有以下几种：  

* EPOLL_CTL_ADD 在epfd中注册指定的fd文件描述符并能把event和fd关联起来。
* EPOLL_CTL_MOD 改变*** fd和evetn***之间的联系。
* EPOLL_CTL_DEL 从指定的epfd中删除fd文件描述符。在这种模式中event是被忽略的，并且为可以等于NULL。  

event这个参数是用于关联制定的fd文件描述符的。它的定义如下：  

```c++
typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;      /* Epoll events */
    epoll_data_t data;        /* User data variable */
};
```

events这个参数是一个字节的掩码构成的。下面是可以用的事件：  

* EPOLLIN - 当关联的文件可以执行 read ()操作时。
* EPOLLOUT - 当关联的文件可以执行 write ()操作时。
* EPOLLRDHUP - (从 linux 2.6.17 开始)当socket关闭的时候，或者半关闭写段的(当使用边缘触发的时候，这个标识在写一些测试代码去检测关闭的时候特别好用)
* EPOLLPRI - 当 read ()能够读取紧急数据的时候。
* EPOLLERR - 当关联的文件发生错误的时候，epoll_wait() 总是会等待这个事件，并不是需要必须设置的标识。
* EPOLLHUP - 当指定的文件描述符被挂起的时候。epoll_wait() 总是会等待这个事件，并不是需要必须设置的标识。当socket从某一个地方读取数据的时候(管道或者socket),这个事件只是标识出这个已经读取到最后了(EOF)。所有的有效数据已经被读取完毕了，之后任何的读取都会返回0(EOF)。
* EPOLLET - 设置指定的文件描述符模式为边缘触发，默认的模式是水平触发。
* EPOLLONESHOT - (从 linux 2.6.17 开始)设置指定文件描述符为单次模式。这意味着，在设置后只会有一次从epoll_wait() 中捕获到事件，之后你必须要重新调用 epoll_ctl() 重新设置。  

返回值：如果成功，返回0。如果失败，会返回-1， errno将会被设置有以下几种错误：  

* EBADF - epfd 或者 fd 是无效的文件描述符。
* EEXIST - op是EPOLL_CTL_ADD，同时 fd 在之前，已经被注册到epoll中了。
* EINVAL - epfd不是一个epoll描述符。或者fd和epfd相同，或者op参数非法。
* ENOENT - op是EPOLL_CTL_MOD或者EPOLL_CTL_DEL，但是fd还没有被注册到epoll上。
* ENOMEM - 内存不足。
* EPERM - 目标的fd不支持epoll。  


## 2.3 等待epoll事件  

```c++
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout);
                      
int epoll_pwait(int epfd, struct epoll_event *events,
                      int maxevents, int timeout,
                      const sigset_t *sigmask);
```

epoll_wait 这个系统调用是用来等待epfd中的事件。events指向调用者可以使用的事件的内存区域。maxevents告知内核有多少个events，必须要大于0.  

timeout这个参数是用来制定epoll_wait 会阻塞多少毫秒，会一直阻塞到下面几种情况：  

1. 一个文件描述符触发了事件。
2. 被一个信号处理函数打断，或者timeout超时。  

当timeout等于-1的时候这个函数会无限期的阻塞下去，当timeout等于0的时候，就算没有任何事件，也会立刻返回。struct epoll_event 如下定义:  

```c++
typedef union epoll_data {
    void    *ptr;
    int      fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;    /* Epoll events */
    epoll_data_t data;      /* User data variable */
};
```

每次epoll_wait() 返回的时候，会包含用户在epoll_ctl中设置的events。  

还有一个系统调用epoll_pwait ()。epoll_pwait()和epoll_wait ()的关系就像select()和 pselect()的关系。和pselect()一样，epoll_pwait()可以让应用程序安全的等待知道某一个文件描述符就绪或者捕捉到信号。下面的 epoll_pwait () 调用：`ready = epoll_pwait(epfd, &events, maxevents, timeout, &sigmask);` 在内部等同于:  

```c++
pthread_sigmask(SIG_SETMASK, &sigmask, &origmask);
ready = epoll_wait(epfd, &events, maxevents, timeout);
pthread_sigmask(SIG_SETMASK, &origmask, NULL);
```

如果 sigmask为NULL, epoll_pwait()等同于epoll_wait()。  

返回值：有多少个IO事件已经准备就绪。如果返回0说明没有IO事件就绪，而是timeout超时。遇到错误的时候，会返回-1，并设置 errno。有以下几种错误:  

* EBADF - epfd是无效的文件描述符
* EFAULT - 指针events指向的内存没有访问权限
* EINTR - 这个调用被信号打断。
* EINVAL - epfd不是一个epoll的文件描述符，或者maxevents小于等于0  

# 三.官方demo  

```c++
#define MAX_EVENTS 10
struct epoll_event  ev, events[MAX_EVENTS];
int         listen_sock, conn_sock, nfds, epollfd;


/* Code to set up listening socket, 'listen_sock',
 * (socket(), bind(), listen()) omitted */

epollfd = epoll_create1( 0 );
if ( epollfd == -1 )
{
    perror( "epoll_create1" );
    exit( EXIT_FAILURE );
}

ev.events   = EPOLLIN;
ev.data.fd  = listen_sock;
if ( epoll_ctl( epollfd, EPOLL_CTL_ADD, listen_sock, &ev ) == -1 )
{
    perror( "epoll_ctl: listen_sock" );
    exit( EXIT_FAILURE );
}

for (;; )
{
    nfds = epoll_wait( epollfd, events, MAX_EVENTS, -1 );
    if ( nfds == -1 )
    {
        perror( "epoll_wait" );
        exit( EXIT_FAILURE );
    }

    for ( n = 0; n < nfds; ++n )
    {
        if ( events[n].data.fd == listen_sock )
        {
            conn_sock = accept( listen_sock,
                        (struct sockaddr *) &local, &addrlen );
            if ( conn_sock == -1 )
            {
                perror( "accept" );
                exit( EXIT_FAILURE );
            }
            setnonblocking( conn_sock );
            ev.events   = EPOLLIN | EPOLLET;
            ev.data.fd  = conn_sock;
            if ( epoll_ctl( epollfd, EPOLL_CTL_ADD, conn_sock,
                    &ev ) == -1 )
            {
                perror( "epoll_ctl: conn_sock" );
                exit( EXIT_FAILURE );
            }
        } else {
            do_use_fd( events[n].data.fd );
        }
    }
}
```



# 四.完整可运行的DEMO  


```c++
#include <stdio.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fcntl.h>
#include <unistd.h>
#include <netdb.h>
#include <errno.h>

#define MAX_EVENT 20
#define READ_BUF_LEN 256

/**
 * 设置 file describe 为非阻塞模式
 * @param fd 文件描述
 * @return 返回0成功，返回-1失败
 */
static int make_socket_non_blocking (int fd) {
    int flags, s;
    // 获取当前flag
    flags = fcntl(fd, F_GETFL, 0);
    if (-1 == flags) {
        perror("Get fd status");
        return -1;
    }

    flags |= O_NONBLOCK;

    // 设置flag
    s = fcntl(fd, F_SETFL, flags);
    if (-1 == s) {
        perror("Set fd status");
        return -1;
    }
    return 0;
}

int main() {
    // epoll 实例 file describe
    int epfd = 0;
    int listenfd = 0;
    int result = 0;
    struct epoll_event ev, event[MAX_EVENT];
    // 绑定的地址
    const char * const local_addr = "192.168.0.45";
    struct sockaddr_in server_addr = { 0 };

    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (-1 == listenfd) {
        perror("Open listen socket");
        return -1;
    }
    /* Enable address reuse */
    int on = 1;
    // 打开 socket 端口复用, 防止测试的时候出现 Address already in use
    result = setsockopt( listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on) );
    if (-1 == result) {
        perror ("Set socket");
        return 0;
    }

    server_addr.sin_family = AF_INET;
    inet_aton (local_addr, &(server_addr.sin_addr));
    server_addr.sin_port = htons(8080);
    result = bind(listenfd, (const struct sockaddr *)&server_addr, sizeof (server_addr));
    if (-1 == result) {
        perror("Bind port");
        return 0;
    }
    result = make_socket_non_blocking(listenfd);
    if (-1 == result) {
        return 0;
    }

    result = listen(listenfd, 200);
    if (-1 == result) {
        perror("Start listen");
        return 0;
    }

    // 创建epoll实例
    epfd = epoll_create1(0);
    if (1 == epfd) {
        perror("Create epoll instance");
        return 0;
    }

    ev.data.fd = listenfd;
    ev.events = EPOLLIN | EPOLLET /* 边缘触发选项。 */;
    // 设置epoll的事件
    result = epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev);

    if(-1 == result) {
        perror("Set epoll_ctl");
        return 0;
    }

    for ( ; ; ) {
        int wait_count;
        // 等待事件
        wait_count = epoll_wait(epfd, event, MAX_EVENT, -1);

        for (int i = 0 ; i < wait_count; i++) {
            uint32_t events = event[i].events;
            // IP地址缓存
            char host_buf[NI_MAXHOST];
            // PORT缓存
            char port_buf[NI_MAXSERV];

            int __result;
            // 判断epoll是否发生错误
            if ( events & EPOLLERR || events & EPOLLHUP || (! events & EPOLLIN)) {
                printf("Epoll has error\n");
                close (event[i].data.fd);
                continue;
            } else if (listenfd == event[i].data.fd) {
                // listen的 file describe 事件触发， accpet事件

                for ( ; ; ) { // 由于采用了边缘触发模式，这里需要使用循环
                    struct sockaddr in_addr = { 0 };
                    socklen_t in_addr_len = sizeof (in_addr);
                    int accp_fd = accept(listenfd, &in_addr, &in_addr_len);
                    if (-1 == accp_fd) {
                        perror("Accept");
                        break;
                    }
                    __result = getnameinfo(&in_addr, sizeof (in_addr),
                                           host_buf, sizeof (host_buf) / sizeof (host_buf[0]),
                                           port_buf, sizeof (port_buf) / sizeof (port_buf[0]),
                                           NI_NUMERICHOST | NI_NUMERICSERV);

                    if (! __result) {
                        printf("New connection: host = %s, port = %s\n", host_buf, port_buf);
                    }

                    __result = make_socket_non_blocking(accp_fd);
                    if (-1 == __result) {
                        return 0;
                    }

                    ev.data.fd = accp_fd;
                    ev.events = EPOLLIN | EPOLLET;
                    // 为新accept的 file describe 设置epoll事件
                    __result = epoll_ctl(epfd, EPOLL_CTL_ADD, accp_fd, &ev);

                    if (-1 == __result) {
                        perror("epoll_ctl");
                        return 0;
                    }
                }
                continue;
            } else {
                // 其余事件为 file describe 可以读取
                int done = 0;
                // 因为采用边缘触发，所以这里需要使用循环。如果不使用循环，程序并不能完全读取到缓存区里面的数据。
                for ( ; ;) {
                    ssize_t result_len = 0;
                    char buf[READ_BUF_LEN] = { 0 };

                    result_len = read(event[i].data.fd, buf, sizeof (buf) / sizeof (buf[0]));

                    if (-1 == result_len) {
                        if (EAGAIN != errno) {
                            perror ("Read data");
                            done = 1;
                        }
                        break;
                    } else if (! result_len) {
                        done = 1;
                        break;
                    }

                    write(STDOUT_FILENO, buf, result_len);
                }
                if (done) {
                    printf("Closed connection\n");
                    close (event[i].data.fd);
                }
            }
        }

    }
    close (epfd);
    return 0;
}
```


# 五.高并发的epoll+多线程Demo
epoll是linux下高并发服务器的完美方案，因为是基于事件触发的，所以比select快的不只是一个数量级。单线程epoll，触发量可达到15000，但是加上业务后，因为大多数业务都与数据库打交道，所以就会存在阻塞的情况，这个时候就必须用多线程来提速。  

下面是来一个网络连接创建一个线程处理业务，业务处理完，线程销毁。实际测试结果不是很理想，在没有业务的时候的测试结果是2000个/s  
测试工具：stressmark  

因为加了适用与ab的代码，所以也可以适用ab进行压力测试。  

```c
char buf[1000] = {0};
sprintf(buf,"HTTP/1.0 200 OK\r\nContent-type: text/plain\r\n\r\n%s","Hello world!\n");
send(socketfd,buf, strlen(buf),0);
```

```c++
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/wait.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <fcntl.h>
#include <sys/epoll.h>
#include <sys/time.h>
#include <sys/resource.h>
#include <pthread.h>

#define MAXBUF 1024 
#define MAXEPOLLSIZE 10000

void* pthread_handle_message(void* para);
/* 
setnonblocking - 设置句柄为非阻塞方式 
*/ 
int setnonblocking(int sockfd) 
{ 
    if (fcntl(sockfd, F_SETFL, fcntl(sockfd, F_GETFD, 0)|O_NONBLOCK) == -1) { 
        return -1; 
    } 
    return 0; 
}

static int count111 = 0;
static time_t oldtime = 0, nowtime = 0;

//------------------------------------------------------------
int main(int argc, char **argv) 
{ 
    int listener, new_fd, nfds, n, ret; 
    struct epoll_event ev; 
    int kdpfd, curfds; 
    socklen_t len; 
    struct sockaddr_in my_addr, their_addr; 
    unsigned int myport, lisnum; 
    struct epoll_event events[MAXEPOLLSIZE]; 
    struct rlimit rt;

    if (argc>1) 
        myport = atoi(argv[1]); 
    else 
        myport = 8006;

    if (argc>2) 
        lisnum = atoi(argv[2]); 
    else 
        lisnum = 10;

    /* 设置每个进程允许打开的最大文件数 */ 
    rt.rlim_max = rt.rlim_cur = MAXEPOLLSIZE; 
    if (setrlimit(RLIMIT_NOFILE, &rt) == -1) { 
        perror("setrlimit"); 
        exit(1); 
    } 
    else printf("设置系统资源参数成功！/n");

    /* 开启 socket 监听 */ 
    if ((listener = socket(PF_INET, SOCK_STREAM, 0)) == -1) { 
        perror("socket"); 
        exit(1); 
    } else 
        printf("socket 创建成功！/n");

    /*设置socket属性，端口可以重用*/ 
    int opt=SO_REUSEADDR; 
    setsockopt(listener,SOL_SOCKET,SO_REUSEADDR,&opt,sizeof(opt));

    /*设置socket为非阻塞模式*/ 
    setnonblocking(listener);

    bzero(&my_addr, sizeof(my_addr)); 
    my_addr.sin_family = PF_INET; 
    my_addr.sin_port = htons(myport); 
    if (argc>3) 
        my_addr.sin_addr.s_addr = inet_addr(argv[3]); 
    else 
        my_addr.sin_addr.s_addr = INADDR_ANY;

    if (bind 
        (listener, (struct sockaddr *) &my_addr, sizeof(struct sockaddr)) 
        == -1) { 
            perror("bind"); 
            exit(1); 
    } else 
        printf("IP 地址和端口绑定成功/n");

    if (listen(listener, lisnum) == -1) { 
        perror("listen"); 
        exit(1); 
    } else 
        printf("开启服务成功！/n");

    /* 创建 epoll 句柄，把监听 socket 加入到 epoll 集合里 */ 
    kdpfd = epoll_create(MAXEPOLLSIZE); 
    len = sizeof(struct sockaddr_in); 
    ev.events = EPOLLIN | EPOLLET; 
    ev.data.fd = listener; 
    if (epoll_ctl(kdpfd, EPOLL_CTL_ADD, listener, &ev) < 0) { 
        fprintf(stderr, "epoll set insertion error: fd=%d/n", listener); 
        return -1; 
    } else 
        printf("监听 socket 加入 epoll 成功！/n"); 
    curfds = 1; 
    while (1) { 
        /* 等待有事件发生 */ 
        nfds = epoll_wait(kdpfd, events, curfds, -1); 
        if (nfds == -1) { 
            perror("epoll_wait"); 
            continue; 
        } 
        /* 处理所有事件 */ 
        for (n = 0; n < nfds; ++n) { 
            if (events[n].data.fd == listener) { 
                new_fd = accept(listener, (struct sockaddr *) &their_addr, 
                    &len); 
                if (new_fd < 0) { 
                    perror("accept"); 
                    continue; 
                } else 
                {
                    //printf("有连接来自于： %s:%d， 分配的 socket 为:%d/n", inet_ntoa(their_addr.sin_addr), ntohs(their_addr.sin_port), new_fd);

                }
                setnonblocking(new_fd); 
                ev.events = EPOLLIN | EPOLLET; 
                ev.data.fd = new_fd; 
                if (epoll_ctl(kdpfd, EPOLL_CTL_ADD, new_fd, &ev) < 0) { 
                    fprintf(stderr, "把 socket '%d' 加入 epoll 失败！%s/n", 
                        new_fd, strerror(errno)); 
                    return -1; 
                } 
                curfds++; 
            } else { 
                pthread_attr_t attr;
                pthread_t threadId;
                

                /*初始化属性值，均设为默认值*/ 
                pthread_attr_init(&attr); 
                pthread_attr_setscope(&attr, PTHREAD_SCOPE_SYSTEM); 
                /* 设置线程为分离属性*/ 
                pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
                if(pthread_create(&threadId,&attr,pthread_handle_message,(void*)&(events[n].data.fd)))
                { 
                    perror("pthread_creat error!"); 
                    exit(-1); 
                } 
            } 
        } 
    } 
    close(listener); 
    return 0; 
}


void* pthread_handle_message(void* para)
{
    char recvBuf[1024] = {0}; 
    int ret = 999;
    int rs = 1;
    int socketfd = *(int *)para;

    while(rs)
    {
        ret = recv(socketfd,recvBuf,1024,0);// 接受客户端消息

        if(ret < 0)
        {
            //由于是非阻塞的模式,所以当errno为EAGAIN时,表示当前缓冲区已无数据可//读在这里就当作是该次事件已处理过。

            if(errno == EAGAIN)
            {
                printf("EAGAIN\n");
                break;
            }
            else{
                printf("recv error! errno:%d\n", errno);
        
                close(socketfd);
                break;
            }
        }
        else if(ret == 0)
        {
            // 这里表示对端的socket已正常关闭. 
            rs = 0;
        }
        if(ret == sizeof(recvBuf))
            rs = 1; // 需要再次读取
        else
            rs = 0;
    }

    if(ret>0){
        count111 ++;
        struct tm *today;
        time_t ltime;
        time( &nowtime );

        if(nowtime != oldtime){
            printf("%d\n", count111);
            oldtime = nowtime;
            count111 = 0;
        }


        char buf[1000] = {0};
        sprintf(buf,"HTTP/1.0 200 OK\r\nContent-type: text/plain\r\n\r\n%s","Hello world!\n");
        send(socketfd,buf, strlen(buf),0);
    }
    close(socketfd);
}

```