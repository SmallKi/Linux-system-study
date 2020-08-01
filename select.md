## select

作用：管理多个文件流，当文件描述符有I/O事件发生时才返回。相当于将阻塞提前了，当read write时一定有数据存在

```c++
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int select(int nfds, fd_set *readfds, fd_set *writefds,
          fd_set *exceptfds, struct timeval *timeout);

//nfds: 最大的文件描述符加1，主要是为了方便select遍历
//readfds：读文件流集合
//writefds：写文件流集合
//exceptfds: 错误文件流集合
//timeout: 设置超时时间，若设置为NULL则整个函数称为阻塞函数

//数据类型说明
//1.fd_set 是一个文件描述符集合
//linux提供了一系列操作它的宏函数
FD_ZERO(FD_SET * set); //将集合清0
FD_SET(int fd, FD_SET* set); //将相应文件描述位置为1
FD_ISSET(int fd, FD_SET* set); //查看相应文件描述符位是否为1
FD_CLR(int fd, FD_SET* set); //将相应文件描述符位置为0

//2. timeval 是时间类型
struct timeval{
  time_t second; //秒
  time_t usecond; //微秒
};
```

select的工作原理

* 当select遍历时，只遍历自己的判断矩阵中为1的那些文件描述符，所以需要提前把要管理的文件描述位通过FD_SET置1
* 当select发现为1的文件描述符发生了I/O事件时，它会将其他的文件描述符都置为0，也就是函数返回后，set集合里只有当前发生了IO事件的文件描述符才为1；
* 用户通过FD_ISSET去遍历所有的文件描述符，当不为0时，说明这个文件描述符有IO操作，用户再根据业务逻辑去执行相关代码

select的注意事项

* 当使用select时，select并不会关注文件描述符的缓冲区，即缓冲区有数据并不会让select发现有文件描述符可读或可写。当我们批量输入时，如果fgets函数一次读取一行的数据，只会接收到第一行，其他行会阻塞住



##### pselect

```c++
#include <sys/select.h>
#include <signal.h>
#include <time.h>
 
int pselect(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timespec *timeout, const sigset_t *sigmask);

//返回：就绪描述字的个数，0－超时，-1－出错
```

与select的区别：

* select函数用的timeout参数，是一个timeval的结构体（包含秒和微秒），然而pselect用的是一个timespec结构体（包含秒和纳秒）
*  select函数可能会为了指示还剩多长时间而更新timeout参数，然而pselect不会改变timeout参数
* select函数没有sigmask参数，当pselect的sigmask参数为null时，两者行为时一致的。有sigmask的时候，pselect相当于如下的select()函数，在进入select()函数之前手动将信号的掩码改变，并保存之前的掩码值；select()函数执行之后，再恢复为之前的信号掩码值。

```c++
sigset_t origmask;
sigprocmask(SIG_SETMASK, &sigmask, &origmask);
select(nfds, &readfds, &writefds, &exceptfds, timeout);
sigprocmask(SIG_SETMASK, &origmask, NULL);
```





## 慢系统调用与EINTR

如write、read、connect、accept这种需要阻塞的系统函数被称为慢系统调用，当慢系统调用被其他的信号发生打断时（如产生了SIGCHILD信号，进程去处理子进程的死亡），信号会产生一个EINTR错误，记录在errno中

但是出现这个错误不代表我们希望程序退出，大多时候我们希望慢系统调用能从终中断处继续执行

解决方法：

* 在信号处理函数中，将sig_flag成员设置为SA_RESTART状态

* 为慢系统调用函数进行特判

  ```c++
  if( (accept(fd, (struct sockaddr*)&sockaddr,&len)) < 0)
  {
      if(errno == EINTR)
          continue;
      else 
      {
          perror("accept");
      	exit(-1);
      }
  }
  
  ```

  注意：这种情况对一些调用是错误的，如connect（或者是一些设置了超时的函数）函数。如果connect函数被打断，不能再次调用这个函数，否则会发生致命错误。因此，当被打断后只能使用select函数来获取到connect函数的返回

  详细内容可以查看 man 7 signal



### 描述符就绪的情况

##### 读就绪

* 该套接字接受缓冲区中的数据字节数大于等于该套接字缓冲区低水位标记（SO_RCVLOWAT）

* 监听套接字且完成的连接数不为0
* 其上有一个套接字错误待处理



##### 写就绪

* 该套接字发送缓冲区中的的可用空间字节数大于等于套接字发送缓冲区低水位标记的当前大小，并且或者该套接字已经连接或不需要连接
* 该连接的写半部关闭
* connect套接字已建立连接，或失败
* 其上有一个套接字错误待处理



## Shutdown函数

作用：将读端或者写端关闭

```c++
       #include <sys/socket.h>

       int shutdown(int sockfd, int how);

//how有三种取值
SHUT_WR  关闭写端
SHUT_RD  关闭读端
SHUT_RDWR  关闭写端和读端
```



## Poll函数

作用：与select相似，也是管理多个文件描述符

```c++
 #include <poll.h>
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

//fds: 是一个struct pollfd结构体的数组
 struct pollfd {
               int   fd;         /* file descriptor */
               short events;     /* requested events */
               short revents;    /* returned events */
           };

//fd设置为-1代表不去检查它的事件
//events 代表想要检查的事件
//revents 有事件发生后，事件会填入这里

//nfds : 数组结构的数量
//timeout ：超时事件，以毫秒为单位，0代表不阻塞，INFTIM(或者-1)代表永远等待即阻塞
```

events的类型

读类型：

* POLLIN      普通或优先级带数据可读（POLLRDNORM & POLLRDBAND）
* POLLRDNORM    普通数据可读
* POLLRDBAND    优先级带数据可读
* POLLPOLLPRI     高优先级数据可读

写类型：

* POLLOUT    普通数据可写
* POLLWRNORM    普通数据可写
* POLLWRBAND   优先级带数据可写

错误事件

* POLLERR
* POLLHUP
* POLLNVAL



## 套接字选项

getsockopt和setsockopt

```c++
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int getsockopt(int sockfd, int level, int optname,
                      void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname,
                      const void *optval, socklen_t optlen);

//sockfd : 套接字描述符
//level： 通用套接字代码
//optname: 要获取或者要设置的参数名
//optval: get用它来获取，set用它作为新值
//optlen: 选项长度
```

套接字的选项可以分为两大类型：

* 标志型：二元选项，启用或不启用。一般用int型，0代表不启用，非0代表启用
* 值类型：设置某些值



几个常用的选项(level 为 SOL_SOCKET)

* SO_REUSEADDR
  * 可以让你启动一个监听服务器，监听相同的地址和端口。
  * 可以让你实现IP别名技术，同一台主机的不同的IP地址绑定相同端口
  * 注意：TCP不允许同地址同端口的别名，不管设没设置这个选项；UDP支持
* SO_KEEPLIVE
  * 如果两个小时没有数据交互，会发送存活探节
* SO_SNDBUF
  * 发送缓冲区
* SO_RCVBUF
  * 接收缓冲区



## Fcntl函数

作用：管理文件描述符的相关控制信息

```c++
#include <unistd.h>
#include <fcntl.h>

int fcntl(int fd, int cmd, ... /* arg */ );
```

设置标志的正确步骤：

* 获取原有的标志

* 将原有的标志|操作形成需要的新标志

* 设置新标志

  ```c++
  int flag;
  flag = fcntl(fd, F_GETFL, 0);
  flag |= O_NOBLOCK;
  fcntl(fd, F_SETFL, flag);
  ```

  