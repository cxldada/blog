---
title: 8.信号
date: 2020-07-30 11:30:09
draft: false
categories: 
- APUE
---

> 信号就是软件中断。
>
> 信号提供了一种异步事件的处理方法
>
> 这篇博文整理的是有关信号机制的综述，并且说明每一种信号的使用方法。并对每一种信号的实现做简要分析。还要分析一下该机制的不足之处。

<!--more-->

## 信号概念

>信号都有一个名字。常常以`SIG`开头
>
>信号的定义一般都放在头文件`<signal.h>`中。其实很多实现都放在了另一个文件，然后让`<signal.h>`包含其中。`Linux`是`<bit/signum.h>`，`FreeBSD`和`MacOs`是`<sys/signal.h>`
>
>`FreeBSD`、`Linux`和`Solaris`支持实时扩展应用程序自定义的信号
>
>不存在编号为0的信号，`kill`函数对编号为0的信号有特殊的应用。`POSIX.1`将此信号称为空信号

### 产生信号的条件

* 当用户按某些终端键时，引发终端产生信号。
* 硬件异常产生信号。比如除数为0、无效内存引用。通常是由硬件检测到，再通知内核，然后内核产生适当的信号
* 进程调用`kill`函数可已将任意信号发送给另一个进程或进程组
* 用户可以调用`kill(可理解为命令)`将任意信号发送给其他进程
* 当检测到某种软件条件已经发生，并应将其通知有关进程时也产生信号

### 处理信号的方式

1. 忽略信号。但是有两种信号决不能被忽略。`SIGKILL`和`SIGSTOP`。原因是：他们向内核和超级用户提供了使进程终止或停止的可靠方法。另外，如果忽略硬件产生的某些信号，则进程的运行行为是未定义的
2. 捕捉信号。当信号发生时，可以指定调用一个用户函数。**注意：不能不能捕捉`SIGKILL`和`SIGSTOP`信号**
3. 执行系统默认动作。

### 信号清单及默认处理方式

|    名字     |                 说明                  |    默认处理     |
| :---------: | :-----------------------------------: | :-------------: |
|  `SIGABRT`  |     异常终止。调用`abort`函数产生     |    终止+core    |
|  `SIGALRM`  | 定时器超时。调用`alarm`函数超时时产生 |      终止       |
|  `SIGBUS`   |               硬件故障                |    终止+core    |
|  `SIGCHLD`  |            子进程状态改变             |      忽略       |
|  `SIGCONT`  |            是暂停进程继续             |    继续/忽略    |
|  `SIGEMT`   |               硬件故障                |    终止+core    |
|  `SIGFPE`   |      算术异常。除以零或浮点溢出       |    终止+core    |
|  `SIGHUP`   |               连接断开                |      终止       |
|  `SIGILL`   |             非法硬件指令              |    终止+core    |
|  `SIGINT`   |              终端中断符               |      终止       |
|   `SIGIO`   |              异步I/O事件              |    终止/忽略    |
|  `SIGIOT`   |               硬件故障                |    终止+core    |
|  `SIGKILL`  |                 终止                  |      终止       |
|  `SIGPIPE`  |      写一个读进程已经关闭的管道       |      终止       |
|  `SIGPROF`  |       更改时间超时(`setitimer`)       |      终止       |
|  `SIGQUIT`  |              终端退出符               |    终止+core    |
|  `SIGSEGV`  |             无效内存引用              |    终止+core    |
|  `SIGSTOP`  |                 停止                  |    停止进程     |
|  `SIGSYS`   |             无效系统调用              |    终止+core    |
|  `SIGTERM`  |                 终止                  |      终止       |
|  `SIGTRAP`  |               硬件故障                |    终止+core    |
|  `SIGTSTP`  |              终端停止符               |    停止进程     |
|  `SIGTTIN`  |             后台读控制tty             |    停止进程     |
|  `SIGTTOU`  |            后台写向控制tty            |    停止进程     |
|  `SIGURG`   |           紧急情况(套接字)            |      忽略       |
|  `SIGUSR1`  |             用户定义信号              |      终止       |
|  `SIGUSR2`  |             用户定义信号              |      终止       |
| `SIGVTALRM` |       虚拟时间时钟(`setitimer`)       |      终止       |
| `SIGWINCH`  |           终端窗口大小改变            |      忽略       |
|  `SIGXCPU`  |       超过CPU限制(`setrlimit`)        | 终止或终止+core |
|  `SIGXFSZ`  |       超过文件长度(`setrlimit`)       | 终止或终止+core |

在下列条件下不会产生core文件：

* 进程是设置用户ID的，而且当前用户并非程序文件的所有者
* 进程是设置组ID的，而且当前用户并非程序文件的组所有者
* 用户没有写当前工作目录的权限
* 文件已存在，而且用户对该文件没有写权限
* 文件太大

## `signal`函数

UNIX系统信号机制最简单的是signal函数

```c
#include <signal.h>
void (*signal(int signo, void (*func)(int)))(int);
// 成功返回以前的信号处理配置，出错返回SIG_ERR
```

* `signo`是信号清单中的信号名
* `func`的值是常量`SIG_IGN`、`SIG_DFL`或当接到此信号后要调用的函数的地址
* `SIG_IGN`表示忽略
* `SIG_DFL`表示默认处理方式

如果觉得上面的signal函数声明很复杂，可以按照下面的拆分形式来理解

```c
typedef void Sigfunc(int);
Sigfunc *signal(int,Sigfunc *);
```

> 当执行一个程序时，所有信号的状态都是系统默认或忽略
>
> 通过练习可以看出，`signal`函数有一个限制：不改变信号的处理方式就不能确定信号的当前处理方式
>
> 当一个进程调用fork时，其子进程继承父进程的信号处理方式
>
> 但是exec函数将原先设为捕捉的信号都改为默认动作。

**因为`signal`函数在各个实现之间的语义不同，所以最好使用后面的`sigaction`函数代替`sinal`函数**

## 不可靠的信号

> 不可靠信号值得是，信号可能会丢失：一个信号发生了，但进程并不知道这一点。同时，进程对信号的控制也很差，无法阻塞一个信号

早期的系统有两个问题：

1. 进程每次接到信号对其进行处理时，随机将该信号动作重置为默认值。导致程序员会再出信号处理程序中再次设置信号处理程序。这里就存在一个时间窗口
2. 用一个变量来记录发生信号时，在变量修改和休眠之间有一个时间窗口

## 中断的系统调用

> 早期UNIX系统的一个特性：如果进程在执行一个低速系统调用而阻塞期间捕捉到一个信号，则该系统调用就被中断不再继续执行
>
> 低速系统调用(指哪些可能使进程永久阻塞的系统调用)包括：
>
> * 如果某些类型文件的数据不存在，则读操作可能会使调用者永久阻塞
> * 如果这些数据不能被相同的类型文件立即接受，则写操作可能会使调用者永久阻塞
> * 在某种条件发生之前打开某些类型文件，可能发生阻塞(例如要打开终端设备，需要等待调制解调器的应答)
> * `pause`函数(休眠直到收到一个信号为止)和`wait`函数
> * 某些`ioctl`操作
> * 某些进程间通信函数
>
> 在低速系统调用中，需要注意的一个例外是与磁盘I/O有关的系统调用。虽然读写一个磁盘文件会暂时阻塞调用者，但是除非发生硬件错误，I/O操作总会很快的返回，并使调用者不再处于阻塞状态
>
> 在4.2`BSD`实现中引入了自动重启动功能：当一个系统调用被一个信号中断时，该系统调用会自动重新调用。目前自动重启动的系统调用包括：`ioctl`、`read`、`readv`、`write`、`writev`、`wait`和`waitpid`。
>
> 如果不希望这些程序这中断后自启动可以使用`sigaciton`函数中的标志进行设置。

## 可重入函数

> 进程捕捉到信号并对其进行处理时，进程正在执行的正常指令序列就被信号处理程序临时中断，它首先执行该信号处理程序中的指令。如果从信号处理程序返回(没有终止或停止)，则继续执行在捕捉到信号时进程正在执行的正常指令。
>
> 但是在信号处理程序中，不能判断捕捉到信号时进程执行到何处。如果进程正在执行`malloc`，在其堆中分配另外的存储空间，而此时由于捕捉信号而插入执行该信号处理程序，其中也有调用`malloc`，可能会对进程数据造成破坏。（因为`malloc`通常为它所分配的存储区维护一个链表，而插入执行信号处理程序时，进程可能正在更改此链表）
>
> 上面这种情况中的`malloc`函数就是不可重入的函数。简单来说就是程序被信号中断时所调用的函数和信号处理程序中不能同时调用的函数，称为不可重入函数，反之就是可重入函数
>
> 大多数函数是不可能重入的，原因如下：
>
> * 它们使用静态数据结构。这样当执行信号处理程序的时候，如果信号处理程序中也调用了，则会覆盖原有程序的数据
> * 他们调用`malloc`和`free`
> * 他们是标准I/O函数。标准I/O库函数的很多实现都不可重入，因为他们使用的是全局数据结构

信号处理期间可能会修改`errno`的值，所以在调用信号处理程序时，一般先保存`errno`，在离开信号处理程序时再恢复`errno`的值

## 可靠信号

> 在信号产生和传递之间有一个时间间隔，在这个时间间隔内我们称这个信号是未决的
>
> 进程可以阻塞信号的递送。如果为进程产生了一个阻塞的信号，而对该信号的动作是系统默认动作或捕捉（不是忽略就行），则为该进程将此信号保持为未决状态，直到该进程对此信号解除阻塞，或者将对此信号的处理动作改为忽略
>
> 如果在进程解除对某个信号的阻塞前，该信号产生了多次的话`POSIX`允许将多次信号进行排队。

## `kill`和`raise`函数

`kill`函数将信号发送给进程或进程组。`raise`函数将信号发送给调用进程

```c
#include <signal.h>
int kill(pid_t pid,int signo);
int raise(int signo);
// 成功返回0，出错返回-1
```

* 调用`raise(signo)`等于调用`kill(getpid(),signo);`
* `kill`的`pid`参数有以下四中情况：
  * `pid > 0`，将信号发送给指定ID的进程
  * `pid == 0`，将信号发送给和调用进程属于同一组的所有进程，而且调用进程必须具有权限
  * `pid < 0`，将信号发送给参数的绝对值指定的进程组，而且调用进程必须具有权限
  * `pid == -1`，将信号发送给调用有权限发送的所有进程
  * 上面的四中情况，调用进程不能向系统进程集中的进程发送信号。系统进程集包含内核进程和`init`进程
* 超级用户可以给任意进程发送信号，非超级用户需要实际用户ID和有效用户ID与接收进程相同。
  * `SIGCONT`信号在发送时进行的权限检测中是一个特例。如果调用进程发送该信号，则可以把这个信号发送给属于同一会话中的任意其他进程
* 如果`signo`为0表示为空信号，可以用来测试一个特定进程是否存在，不过意义不大
* 如果调用`kill`为调用进程产生信号，而且此信号是不被阻塞的，**那么在`kill`返回之前**，`signo`或者某个其他未决的、非阻塞信号被传送至该进程。

## `alarm`和`pause`函数

使用`alarm`函数可以设置一个定时器，在将来某个时刻该定时器会超时。当定时器超时时，会产生`SIGALRM`信号。系统默认操作是终止该调用进程。

```c
#include <unistd.h>
unsigned int alarm(unsigned int seconds);
// 返回0或者以前设置的闹钟时间剩余秒数
```

* 每个进程只有一个闹钟时间。
  * 如果在调用`alarm`时，之前已为该进程注册的闹钟时间还没有超时，则该闹钟时间剩余秒数作为本次`alarm`的返回值。以前注册的闹钟时间被新值代替
  * 如果有以前尚未超时的闹钟时间，且本次参数为0，则取消以前设置的闹钟时间，剩余值为函数返回值

`pause`函数使进程挂起直至捕捉到一个信号

```c
#incldue <unistd.h>
int pause(void);
// 返回值-1，errno设置为EINTR
```

* 只有执行了一个信号处理程序并从其返回时，`pause`才返回。

下面要结合几个实例来分析实际使用中，可能存在的问题

### 实现sleep功能

#### 实例1

```c
#include <signal.h>
#include <unistd.h>
static void sig_alrm(int signo)
{
    // 什么都不做，直接返回为了唤醒pause
}

unsigned int sleep1(unsigned int seconds)
{
    if(signal(SIGALRM,sig_alrm) == SIG_ERR)
        return secodes;
    alarm(seconds);
    pause();
    return alram(0);
}
```

在这个实例中有一下三个问题

1. 如果在调用`sleep1`之前，调用者设置了闹钟，则它会被`sleep1`函数中第一次`alarm`调用擦除
2. 该程序中修改了对`SIGALRM`的配置。如果编写了一个函数供其他函数调用，则在该函数被调用时先要保存原有配置，在该函数返回前再恢复原配置
3. 在第一次调用`alarm`和`pause`之间有一个竞争条件。在一个繁忙的系统中，可能`alarm`在调用`pause`之前超时，并调用了信号处理程序，则在在调用`pause`之后，可能会造成永久挂起

#### 实例2

这个实例用来解决实例1中的第三个问题

```c
#include <setjmp.h>
#include <signal.h>
#include <unistd.h>

static jmp_buf env_alrm;

static void sig_alrm(int signo)
{
    longjmp(env_alrm,1);
}

unsigned int sleep2(unsigned int seconds)
{
    if(signal(SIGALRM,sig_alrm) == SIG_ERR)
        return secondes;
    if(setjmp(env_alrm) == 0)
    {
        alarm(seconds);
        pause();
    }
    return alarm(0);
}
```

此函数中，避免了实例1中的竞争条件问题。即使`pause`从未执行，在发生`SIGALRM`时，`sleep2`已经返回

但是，`sleep2`函数中还有一个难以察觉的问题，它涉及与其他信号的交互。如果`SIGALRM`中断了某个其他信号处理程序，则调用`longjmp`会提早终止该信号处理程序。运行下面的例子就会发现这个问题

```c
#include "../include/apue.h"
#include <setjmp.h>
#include <signal.h>
#include <unistd.h>

static jmp_buf env_alrm;
static void sig_int(int);

static void sig_alrm(int signo)
{
    longjmp(env_alrm, 1);
}

unsigned int sleep2(unsigned int seconds)
{
    if (signal(SIGALRM, sig_alrm) == SIG_ERR)
        return seconds;
    if (setjmp(env_alrm) == 0)
    {
        alarm(seconds);
        pause();
    }
    return alarm(0);
}

int main(int argc, char const *argv[])
{
    unsigned int unslept;
    if (signal(SIGINT, sig_int) == SIG_ERR)
        err_sys("signal(SIGINT) error");
    unslept = sleep2(5);
    printf("sleep2 returned: %u\n", unslept);

    return 0;
}

static void sig_int(int signo)
{
    int i, j;
    volatile int k;

    printf("\nsig_int starting\n");
    for (i = 0; i < 300000; i++)
        for (j = 0; j < 4000; j++)
            k += i * j;
    printf("sig_int finished\n");
}
```

运行这个程序后，你会发现，当你在程序开始执行时，键入中断字符(`Ctrl+c`)，`SIGINT`的处理程序还没有执行完，而`SIGALRM`的处理程序执行完了，导致整个程序提前结束

总之，在处理信号时一定要仔细一点

## 信号集

我们需要一个能表示多个信号的数据类型，以便告诉内核不允许发生该信号集中的信号。

`POSIX.1`定义了`sigset_t`类型来包含一个信号集，操作信号集的函数有下列5个：

```c
#include <signal.h>
int sigemptyset(sigset_t *set);
int sigfillset(sigset_t *set);
int sigaddset(sigset_t *set);
int sigdelset(sigset_t *set);
// 这四个函数若成功返回0，出错返回-1
int sigismember(const sigset_t *set,int signo);
// 若真，返回1，若假，返回0
```

* `sigemptyset`初始化由`set`指向的信号集，清除所有信号
* `sigfillset`初始化由`set`指向的信号集，使其包括所有信号
* 使用前一定要调用`sigemptyset`或`sigfillset`一次，来进行初始化
* 对所有以信号集作为参数的函数，总是以信号集地址作为向其传送的参数

## 函数`sigprocmask`

该函数可以检测或更改进程的信号屏蔽字

```c
#include <signal.h>
int sigprocmask(int how,const sigset_t *restrict set,sigset_t *restrict oset);
// 成功返回0，出错返回-1
```

* 若`oset`是一个非空指针，那么进程的当前信号屏蔽字将通过该值返回
* 若`set`是一个非空指针，则`how`指定了如何修改当前信号屏蔽字
  * `how`为`SIG_BLOCK`，则新的屏蔽字是当前屏蔽字与`set`参数并集的结果。`set`参数包含了希望阻塞的信号
  * `how`为`SIG_UNBLOCK`，则新的屏蔽字是当前屏蔽字与`set`参数的补集的交集。`set`参数包含了希望解除阻塞的信号
  * `how`为`SIG_SETMASK`，则当前信号屏蔽字直接设置为`set`
* 若`set`是空指针，则不改变进程的信号屏蔽字，`how`参数无意义
* 在调用`sigprocmask`后如果有任何未决的、不再阻塞的信号，则在`sigprocmask`返回前，至少将其中之一递送给该进程

## 函数`sigpending`

该函数返回一个信号集，对调用进程而言，其中的各个信号是阻塞不能递送的，因而也是当前未决的

```c
#include <signal.h>
int sigpending(sigset_t *set);
//成功返回0，出错返回-1
```

## 函数`sigaction`

`sigaction`函数的功能是检查或修改与执行信号相关联的处理动作。

```c
#include <signal.h>
int sigaction(int signo,const struct sigaction *restrict act,struct sigaction *restrict oact);
// 成功返回0，出错返回-1
```

* `signo`，是要检测或修改其具体动作的信号编号
* 若`act`指针非空，则表示要修改的动作
* 如果`oact`指针非空，则系统经由`oact`指针返回该信号的上一个动作

`struct sigaction`结构如下

```c
struct sigaction {
    void (*sa_handler)(int);	// 要添加的信号处理函数
    sigset_t sa_mask;			// 要添加的信号屏蔽字(SIG_BLOCK方式添加)
    int sa_flags;				// 操作选项
    void (*sa_sigaction)(int,siginfo_t *,void *); // 当sa_flags=SA_SIGINFO时有用
};
```

* `sa_handler`字段包含一个信号捕捉函数的地址，不是`SIG_IGN`或`SIG_DFL`

* `sa_mask`字段说明了一个信号集，在调用该信号捕捉函数之前，这一信号集要加到进程的信号屏蔽字中。

  * 仅当从信号捕捉函数返回时在将进程的新号屏蔽字恢复为原先值
  * 保证了在处理一个给定信号时，如果这种信号再次发生，那么它会被阻塞到对前一个信号的处理结束为止

* `act`结构的`sa_flags`字段指定对信号进行处理的各个选项

  * |      选项      |                             说明                             |
    | :------------: | :----------------------------------------------------------: |
    | `SA_INTERRUPT` |              由此信号中断的系统调用不自动重启动              |
    | `SA_NOCLDSTOP` | 若`signo`是`SIGCHLD`，当子进程停止，不产生此信号。当子进程终止时仍产生此信号。若已设置此标志，则当停止的进程继续运行时，作为`XSI`扩展，不产生该信号 |
    | `SA_NOCLDWAIT` | 若`signo`是`SIGCHLD`，则当调用进程的子进程终止时，不创建僵死进程。若调用进程随后调用`wait`，则阻塞到它所有子进程都终止，此时返回-1，`errno`设置为`ECHILD` |
    |  `SA_NODEFER`  | 当捕捉到此信号时，在执行其信号捕捉程序时，系统不自动阻塞此信号(除非`sa_mask`中包含了此信号)。 |
    |  `SA_ONSTACK`  | 若`sigaltstack`已声明了一个替换栈，则此信号递送给替换栈上的进程 |
    |  `SA_RESTART`  |              由此信号终端的系统调用会自动重启动              |
    | `SA_RESETHAND` | 在此信号捕捉函数的入口处，将此信号的处理方式重置为`SIG_DFL`,并清除`SA_SIGINFO`标志。 |
    |  `SA_SIGINFO`  | 此选项对信号处理程序提供了附加信息：一个指向`siginfo`结构的指针以及一个指向上下文标识符的指针 |

* `sa_sigaction`字段是一个替代的信号处理程序，在`sigaction`结构中使用`SA_SIGINFO`标志时，使用该信号处理程序。

  * 对于`sa_sigaction`和`sa_handler`字段，实现可能使用同一存储区，所以信号处理程序使用二者之一

  * `siginfo`结构包含了信号产生原因的相关信息。结构大致如下

    * ```c
      struct siginfo{
          int si_signo; // POSIX 要求至少包含此成员
          int si_errno;
          int si_code; // POSIX 要求至少包含此成员
          pid_t si_pid;
          uid_t si_uid;
          void *si_addr;
          int si_status;
          union sigval si_value;
      };
      
      union sigval {
          int saval_int;
          void *sival_pt;
      };
      ```

## 函数`sigsetjmp`和`siglongjmp`

在信号处理程序中进行非局部转移时应当使用下面这两个函数

```c
#include <setjmp.h>
int sigsetjmp(sigjmp_buf env,int savemask);
// 直接调用返回0，从siglongjmp调用返回，则返回非0
void siglongjmp(sigjmp_buf env,int val);
```

这两个函数和`setjmp`、`longjmp`之间唯一的区别是`sigsetjmp`增加了一个参数。

* 如果`savemask`非0，则`sigsetjmp`在`env`中保存进程的当前信号屏蔽字
* 调用`siglongjmp`时，如果带非0`savemask`的`sigsetjmp`调用已经保存了`env`，则`siglongjmp`从其中恢复保存的信号屏蔽字

## 函数`sigsuspend`

> 如果希望对一个信号解除阻塞，然后`pause`以等待以前被阻塞的信号发生。则会出现一种情况：在解除阻塞和`pause`之间有一个时间窗口，如果在时间窗口中产生了信号，而且该信号只产生一次，则`pause`可能会造成永久阻塞

为了解决上面这种情况，就需要把解除阻塞和`pause`作为一个原子操作，而`sigsuspend`正是这样的功能

```c
#include <signal.h>
int sigsuspend(const sigset_t *sigmask);
// 返回值-1，并将errno设置为EINT
```

* 进程的信号屏蔽字设置为由`sigmask`指向的值
* 在捕捉到一个信号或发生了一个会终止该进程的信号之前，该进程被挂起。
* 如果捕捉到一个信号而且从该信号处理程序返回，则`sigsuspend`返回，并且将该进程的信号屏蔽字设置为调用之前的值
* 此函数没有成功返回。

## 函数`abort`

该函数的功能是使程序异常终止

```c
#include <stdlib.h>
void abort(void);
// 此函数没有返回值
```

* 此函数将`SIGABRT`信号发送给调用进程。
* 让进程捕捉`SIGABRT`的意图是：在进程终止之前由进程自己决定执行所需的清理操作。
* 如果进程设置的信号处理程序未关闭自己，则当处理程序返回时，`abort`会终止该进程
* `POSIX.1`要求：如果`abort`调用终止进程，则它对所有打开标准I/O流的效果应当与进程终止前对每个流调用`fclose`相同

## System函数的实现

在学习进程控制原语时，使用`exec`函数实现过一版`system`函数，但是那一版的实现是没有处理信号的。`POSIX`要求`system`函数忽略`SIGINT`和`SIGQUIT`信号，并且阻塞`SIGCHLD`信号。

```c
#include	<sys/wait.h>
#include	<errno.h>
#include	<signal.h>
#include	<unistd.h>

int
system(const char *cmdstring)	/* with appropriate signal handling */
{
	pid_t				pid;
	int					status;
	struct sigaction	ignore, saveintr, savequit;
	sigset_t			chldmask, savemask;

	if (cmdstring == NULL)
		return(1);		/* always a command processor with UNIX */

	ignore.sa_handler = SIG_IGN;	/* ignore SIGINT and SIGQUIT */
	sigemptyset(&ignore.sa_mask);
	ignore.sa_flags = 0;
	if (sigaction(SIGINT, &ignore, &saveintr) < 0)
		return(-1);
	if (sigaction(SIGQUIT, &ignore, &savequit) < 0)
		return(-1);
	sigemptyset(&chldmask);			/* now block SIGCHLD */
	sigaddset(&chldmask, SIGCHLD);
	if (sigprocmask(SIG_BLOCK, &chldmask, &savemask) < 0)
		return(-1);

	if ((pid = fork()) < 0) {
		status = -1;	/* probably out of processes */
	} else if (pid == 0) {			/* child */
		/* restore previous signal actions & reset signal mask */
		sigaction(SIGINT, &saveintr, NULL);
		sigaction(SIGQUIT, &savequit, NULL);
		sigprocmask(SIG_SETMASK, &savemask, NULL);

		execl("/bin/sh", "sh", "-c", cmdstring, (char *)0);
		_exit(127);		/* exec error */
	} else {						/* parent */
		while (waitpid(pid, &status, 0) < 0)
			if (errno != EINTR) {
				status = -1; /* error other than EINTR from waitpid() */
				break;
			}
	}

	/* restore previous signal actions & reset signal mask */
	if (sigaction(SIGINT, &saveintr, NULL) < 0)
		return(-1);
	if (sigaction(SIGQUIT, &savequit, NULL) < 0)
		return(-1);
	if (sigprocmask(SIG_SETMASK, &savemask, NULL) < 0)
		return(-1);

	return(status);
}
```

### system的返回值

在编写使用`system`函数的程序时，一定要正确地解释返回值。如果直接调用`fork`、`exec`和`wait`，则终止状态与调用`system`是不同的。

## sleep、nanosleep和clock_nanosleep函数

函数的定义

```c
#include <unistd.h>
unsigned int sleep(unsigned int seconds);
// 返回0或未休眠完的秒数

#include <time.h>
int nanosleep(const struct timespec *reqtp, struct timespec *remtp);
// 若休眠到要求的时间返回0，如出错返回-1

#include <time.h>
int clock_nanosleep(clockid_t clock_id, int flags,
                   const struct timespec *reqtp,
                   struct timespec *remtp);
// 若休眠到要求的时间返回0，如出错返回-1
```

* 这三个函数会使得调用进程被挂起直到满足下列两个条件之一：
  1. 已经过了指定的时间
  2. 调用进程捕捉到一个信号并从信号处理程序返回
* 后两个函数提供了更高的时间精度，并且他们都是通过最后一个参数来返回未休眠完的时间值。
* 关于最后一个函数
  * `flags`参数为0表示休眠时间是相对的
  * `flags`参数为`TIMER_ABSTIME`，表示休眠时间是绝对的

## sigqueue函数

使用排队信号必须做以下几个操作：

* 使用`sigaction`函数安装信号处理程序时指定`SA_SIGINFO`标志。如果没有给出这个标志，信号会延迟，但信号是否进入队列要取决于具体实现
* 在`sigaction`结构的`sa_sigaction`成员中提供信号处理程序。实现可能允许用户使用`sa_hanlder`字段，但不能获取`sigqueue`函数发送出来的额外信息
* 使用`sigqueue`函数发送信号，而不是使用`kill`

函数定义

```c
#include <signal.h>
int sigqueue(pid_t pid,int signo,const union sigval value);
// 成功返回0，出错返回-1
```

* 这个函数只能把信号发送给单个进程，可以使用`value`参数向信号处理程序传递整数和指针值，除此之外，`sigqueue`函数与`kill`函数类似
* 信号不能被无限排队，由`SIGQUEUE_MAX`限制。达到相应的限制以后，`sigqueue`就会失败，将`errno`设为`EAGAIN`

## 信号名和编号

某些系统提供数组`extern char *sys_siglist[];`数组下标是信号编号，数组中的元素是指向信号名的字符串指针

可以使用`psignal`函数可移植地打印与信号编号对应的字符串

```c
#include <signal.h>
void psignal(int signo,const char *msg);
```

* 字符串`msg`输出到标准错误文件，后面跟随一个冒号和一个空格，在后面是对该信号的说明，最后一个换行符
* 如果`msg`为`NULL`，则只有信号说明部分

如果在`sigaction`信号处理程序中有`siginfo`结构，可以使用`psiginfo`函数打印信号信息

```c
#include <signal.h>
void psiginfo(const siginfo_t *info,const char *msg);
```

* 这个函数的在不同平台输出的信息可能有所不同

如果只需要信号的字符描述部分，也不需要把它写到标准错误文件中，可以使用`strsignal`函数

```c
#include <string.h>
char *strsignal(int signo);
// 返回指向描述该信号的字符串指针
```

## 尾巴

> 信号这一章的内容很多，有些不好理解，不好调试。多实例代码去验证，要注意不同平台之间的差别。在编写信号相关的程序时，一定要细心，有时不会出错，但在繁忙的系统中就有可能出错。
>
> 好的建议，请发送建议或修改内容到邮箱：c.x.l@live.com
>
