---
title: 13.进程间通信
date: 2020-07-30 11:30:33
draft: false
categories: 
- APUE
---

## 简介

> 前段时间有点忙，不管是工作还是生活。一直没有抽出完整的时间来学习`《APUE》`的剩余章节。
>
> 现在事情基本都处理完了，我想可以继续记录我的学习历程了
>
> 这篇博文主要记录关于进程间通信所用的相关技术。
>
> 进程间通信`（InterProcess Communication,IPC）`
>
> UNIX中常见的`IPC`类型
>
> * 半双工管道
> * FIFO
> * 全双工管道
> * 命名全双工管道
> * `XSI`消息队列
> * `XSI`信号量
> * `XSI`共享存储
> * 消息队列（实时）
> * 信号量
> * 共享存储（实时）
> * 套接字
> * STREAMS
>
> 下面是详细内容记录

<!--more-->

## 管道

> 管道是UNIX系统`IPC`的最古老形式，所有的UNIX系统都提供此种通信机制。
>
> 不过管道有两个局限性：
>
> 1. 它们都是半双工的（数据只能从一个方向上流动）。虽然现在有些系统提供双全工管道，但是为了最佳的可移植性，我们绝对不能预先假定系统支持全双工管道
> 2. 管道只能在具有公共祖先的两个进程间使用
>
> 管道最常用的地方就是shell了，想shell中输入一个命令序列后，shell会为每一条命令创建一个进程，然后用管道将前一条命令的标准输入与后一条命令的标准输入相连接。

### 创建管道的函数

```c
#include <unistd.h>
int pipe(int fd[2]);
// 成功返回0，出错返回-1
```

* 通过参数`fd`返回的两个文件描述如有如下说明：
  * `fd[0]`为读而打开
  * `fd[1]`为写而打开
  * `fd[1]`的输出是`fd[0]`的输入
* 对于支持全双工管道的系统来说，`fd[0],fd[1]`都是以读写方式打开的
* 管道的一般使用方式：进程先调用`pipe`函数，接着调用`fork`函数，这样就创建了从父进程到子进程的`IPC`通道

当管道的一端被关闭后，下面两个规则会生效：

1. 当读(read)一个写端关闭的管道时，所有数据都被读取后，read返回0，表示文件结束。通常情况下，如果管道的写端还有进程，就不会产生文件的结束
2. 如果写(write)一个读端关闭的管道时，会产生信号`SIGPIPE`。如果忽略该信号或者捕捉该信号并从其处理程序返回，则write返回-1，`errno`被设置为`EPIPE`

在写管道(或者FIFO)时，常量`PIPE_BUF`规定了内核的管道缓冲区大小。如果管道调用write，而且要求写的字节数小于等于`PIPE_BUF`，则该写操作不会与其他进程对同一管道的write操作交叉进程。如果超过了限制，则写的数据可能会相互交叉。

用`pathconf`或者`fpathconf`函数可以确定`PIPE_BUF`的值

## 函数`popen`和`pclose`

> 这两个函数实现的操作是：创建一个管道，fork一个子进程，关闭未使用的管道端，执行一个shell运行命令，然后等待命令终止

```c
#include <stdio.h>
FILE *popen(const char *cmdstring,const char *type);
// 成功返回文件指针，出错返回NULL
int pclose(FILE *fp);
// 成功返回cmdstring的终止状态，出错返回-1
```

* `type`为`r`时，`popen`函数返回的文件指针连接到`cmdstring`的标准输出，为`w`时，则连接的是标准输入
* `pclose`函数会关闭标准I/O流，等待命令终止，然后返回shell的终止状态

## 协同进程

> UNIX系统过滤程序从标准输入读取数据，向标准输出写数据。
>
> 几个过滤程序通常在shell管道中线性连接。
>
> 当一个过滤程序即产生某个过滤的输入，有读取该过滤程序的输出时，它就变成了协同进程。

## 消息队列

> 消息队列是消息的链接表，存储在内核中，由消息队列标识符标识。
>
> `msgget`用于创建一个新队列或打开一个现有队列
>
> `msgsnd`将新消息添加到队列尾端。每个消息包含一个正的长整型类型的字段、一个非负的长度以及实际数据字节数。
>
> `msgrcv`用于从队列中取消息。我们并不一定要以先进先出的顺序取消息，也可以按照消息的类型字段取消息

每个队列都有一个`msqid_ds`结构与其相关联，结构内容大致如下：

```c
struct msqid_ds {
    struct ipc_perm msg_perm;	// 见下
    msgqnum_t msg_qnum;		// 在队列中的消息数
    msglen_t msg_qbytes;	// 队列中的最大字节数
    pid_t msg_lspid;	// 最后一次调用msgsnd()的pid
    pid_t msg_lrpid;	// 最后一次调用msgrcv()的pid
    time_t msg_stime;	// 最后调用msgsnd()的时间
    time_t msg_rtime;	// 最后调用msgrcv()的时间
    time_t msg_ctime;	// 最后修改的时间
    // ........  还有一些是根据实现不同而自定的字段
};

struct ipc_perm {
    uid_t uid;		// 拥有者的id
    git_t gid;		// 拥有者所属的组ID
    uid_t cuid;		// 创建者的id
    git_t cgid;		// 创建者所属的组ID
    mode_t mode;	// 访问权限
    // ...... 还有一些是根据实现不同而自定义的字段
};
```

### 创建或者打开消息队列的函数

```c
#include <sys/msg.h>
int msgget(key_t key,int flag);
// 成功返回消息队列ID，出错返回-1
```

> 该函数的返回值是消息队列的标识符，这个标识符是IPC对象的内部名。为使多个合作进程能够在同一IPC对象上汇聚，需要提供一个外部命名方案。为此每个IPC对象都与一个键相关联，这个键就是IPC对象的外部名。
>
> 在该函数中参数key就是我们要指定的IPC对象的外部名。

有多种方法是客户进程和服务器进程在同一个IPC结构上汇聚：

1. 服务器进程可以指定键为`IPC_PRIVATE`来创建一个新IPC结构，将返回的标识符存放在某处(如一个文件)以便客户进程取用。
   1. 这种方法有个缺点：服务器进程需要将队列表示写入文件中，客户进程又要读这个文件来获得此标识符
2. 可以在一个公用头文件中定义一个客户进程和服务器进程都认可的键。然后服务器进程指定此键创建一个新的IPC结构。
3. 客户进程和服务器进程认同一个路径名和项目ID(0～255)之间的字符值，接着，调用`frok`函数将这两个值变换为一个键。然后再方法2中使用此键

```c
#include <sys/ipc.h>
key_t ftok(const char *path,int id);
// 成功返回键，出错返回(key_t)-1
```

* `path`参数必须引用一个现有的文锦啊。产生键时，只使用`id`参数的低8位

使用`msgget`函数创建新队列时，要舒适化`msqid_ds`结构的下列成员：

* ipc_perm结构。其中的mode成员安flag中相应的权限位设置
* `msg_qnum`、`msg_lspid`、`msg_lrpid`、`msg_stime`和`msg_ctime`都设置为0
* `msg_ctime`被设置为当前时间
* `msg_qbytes`设置为系统限制值

### 消息队列垃圾桶函数

```c
#include <sys/msg.h>
int msgctl(int msqid,int cmd,struct msgqid_ds *buf);
// 成功返回0，出错返回-1
```

* `cmd`参数指定对`msqid`指定的队列要执行的命令
  * `IPC_STAT`取此队列的`msqid_ds`结构，并将它存放在`buf`指向的结构中
  * `IPC_SET`将字段`msg_perm.uid`、`msg_perm.gid`、`msg_perm.mode`和`msg_qbutes`从`buf`指向的结构复制到与这个队列相关联的msqid_ds`中。
    * 只能由`msgqid_ds`结构的拥有者和root账户使用。
  * `IPC_RMID`从系统中删除该消息队列以及仍在队列中的所有数据。这种删除立即生效
* `cmd`的三条命令也可以用于后面记录的信号量和共享存储

### 将数据放到消息队列的函数

```c
#include <sys/msg.h>
int msgsnd(int msgid,const void *ptr,size_t nbytes,int flag);
// 成功返回0，出错返回-1
```

* `ptr`参数可以指向一个长整型数（表示消息类型），后面跟着实际的消息数据，结构可以如下：

```c
struct mymesg {
    long mtype;			// 消息类型
    char mtext[512];	// 消息数据
};
```

* `flag`参数为`IPC_NOWAIT`时，表示不阻塞

### 从队列取消息的函数

```c
#include <sys/msg.h>
ssize_t msgrcv(int msqid,void *ptr,size_t nbytes,long type,int flag);
// 成功返回消息数据部分的长度。出错返回-1
```

* `ptr`参数与`msgsnd`中的一样
* `nbytes`指数据缓冲区的长度。
* 如果返回的消息长度大于`nbytes`，而且`flag`中设置了`MSG_NOERROR`位，则该消息会被截断。如果没有则会出错返回
* `type`指定想要那种类型的消息
  * `type==0`返回队列中的第一个消息
  * `type > 0`返回消息类型为`type`的第一个消息
  * `type < 0`返回队列中消息类型值小于等于`type`绝对值的消息，如果有若干个，则取值最小的消息
* `msgrcv`执行陈宫时，内核会自动更新与该消息队列相关联的`msgid_ds`结构，以指示调用者的进程ID和调用时间，并指示队列中的消息数减少了一个

建议： 在编写新的应用程序是不应当在使用消息队列了，因为它相比其他`IPC`机制没有什么优势了

## 信号量
**待续**
