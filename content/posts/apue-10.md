---
title: 10.线程控制
date: 2020-07-30 11:30:19
draft: false
categories: 
- APUE
---

> 前面讲学习了线程以及线程同步的基础知识。但是对于线程相关的属性和线程同步的属性没有描述，这里主要学习控制线程行为方面的内容，介绍线程属性和同步原语属性。
>
> 还要学习同一个进程中的多个线程之间如何保持数据的私有性。最后还描述了基于进程的系统调用如何与线程进行交互

<!--more-->

## 线程属性

管理线程属性的函数都遵循相同的模式：

1. 每个对象与它自己类型的属性对象进行关联(线程与线程属性关联，互斥量与互斥量属性关联，等等)
2. 有一个初始化函数，把属性设为默认值
3. 一个销毁属性对象的函数
4. 每个属性都有一个从属性对象获取属性值得函数
5. 每个属性都有一个设置属性值得函数

线程属性使用的`pthread_attr_t`数据结构来表示。该结构的初始化和销毁函数如下

```c
#include <pthread.h>
int pthread_attr_init(pthread_attr_t *attr);
int pthread_attr_destroy(pthread_attr_t *attr);
// 成功返回0，否则返回错误编号
```

下表总结了线程属性的值

|    属性值     |            描述            |
| :-----------: | :------------------------: |
| `detachstate` |      线程分离状态属性      |
|  `guardsize`  | 线程栈末尾的警戒缓冲区大小 |
|  `stackaddr`  |      线程栈的最低地址      |
|  `stacksize`  |  线程栈的最小长度(字节数)  |

下面的函数用来获取和设置线程属性中`detachstate`的值

```c
#include <pthread.h>
int pthread_attr_getdetachstate(const pthread_attr_t *restict attr,
                                int *detachstate);
int phtread_attr_setdatechstate(pthread_attr_t *attr,int *detachstate);
// 成功返回0，出错返回错误编号
```

* 第二个参数可以设置为`PTHREAD_CREATE_DETACHED`。表示已分离状态启动线程
* 第二个参数也可以设置为`PTHREAD_CREATE_JOINABLE`。表示正常启动线程

------------------------

对于遵循`POSIX`标准的操作系统，并不一定支持线程栈属性，但是遵循`Single UNIX specification`的系统，是必须支持线程栈属性的。

* 在编译期间可以使用`_POSIX_THREAD_ATTR_STACKADDR`和`_POSIX_THREAD_ATTR_STACKSIZE`符号来检查系统是否支持线程栈属性
* 也可以在运行期间以`_SC_THREAD_ATTR_STACKADDR`和`_SC_THREAD_ATTR_STACKSIZE`作为参数传给`sysconf`函数，来检测是否支持线程栈属性

```c
#include <pthread.h>
int pthread_attr_getstack(const pthread_attr_t *restrict attr,
                         void **restrict stackaddr,
                         size_t *restrict stacksize);
int pthread_attr_setstack(pthread_attr_t *attr,void *stackaddr,size_t stacksize);
// 成功返回0，出错返回错误编号
```

* 对于**进程**来说，虚地址空间的大小是固定的。但是对于线程来说，同样大小的虚地址空间必须被所有的线程栈共享。
  * 如果应用程序使用了许多线程，以致这些线程栈的累计大小超过了可用的虚地址空间，就需要减少默认的线程栈大小。
  * 另一方面，如果线程调用的函数分配了过多的自动变量，或者调用的函数涉及许多很深的栈帧，那么需要的栈大小可能要比默认的大
* `stackaddr`线程属性被定义为栈的最低内存地址，但这并不一定是栈的开始位置。因为有些实现是从高地址向低地址增长，有些则相反

程序还可以通过调用下面两个函数来设置线程栈的大小

```c
#include <pthread.h>
int pthread_attr_getstacksize(const pthread_attr_t *restrict attr,
                             size_t *restrict stacksize);
int pthread_attr_setstacksize(const pthread_attr_t *attr,size_t stacksize);
// 成功返回0，出错返回错误编号
```

* `stacksize`的值不能小于`PTHREAD_STACK_MIN`

线程属性`guardsize`控制着线程栈末尾之后用来避免栈溢出的扩展内存的大小。这个属性的默认值通常是系统页大小。

把`guardsize`设置为0，表示不允许发生这个特征的行为：这种情况下，不会提供警戒缓冲区

可以使用下面两个函数来获取和设置该属性

```c
#include <pthread.h>
int pthread_attr_getguardsize(const pthread_attr_t *restrict attr,
                             size_t *restrict guardsize);
int pthread_attr_setguardsize(const pthread_attr_t *attr,size_t guardsize);
// 成功返回0，出错返回错误编号
```

* 如果该属性被修改了，操作系统可能会把它取为页大小的整数倍
* 如果线程的栈指针溢出到警戒区域，程序就可能通过信号接收到错误信息

还有一些其他的线程属性，《APUE》没有进行讨论，需要自己补充。不过线程中还有一部分属性是`pthread_addr_t`中没有的：可撤销状态和可撤销类型。

## 同步属性

### 互斥量属性

互斥量属性用`pthread_mutexattr_t`结构表示。通常是使用`PTHREAD_MUTEX_INITIALIZER`常量或者指向互斥量属性结构的空指针作为参数调用`pthread_mutex_init`函数

初始化和销毁互斥量属性的函数

```c
#include <pthread.h>
int pthread_mutexattr_init(pthread_mutexattr_t *attr);
int pthread_mutexattr_destroy(pthread_mutexattr_t *attr);
// 成功返回0，出错返回错误编号
```

关于互斥量属性，值得关注的是一下三个属性：

1. 进程共享属性。可以设置的只有：
   1. `PTHREAD_PROCESS_PRIVATE`(默认值)
   2. `PTHREAD_PROCESS_SHARED`
2. 健壮属性。有两种值可以设定：
   1. `PTHREAD_MUTEX_STALLED`(默认值)
   2. `PTHREAD_MUTEX_ROBUST`
3. 类型属性。共有四个值：
   1. `PTHREAD_MUTEX_NORMAL`
   2. `PTHREAD_MUTEX_ERRORCHECK`
   3. `PTHREAD_MUTEX_RECURSICE`
   4. `PTHREAD_MUTEX_DEFAULT`

------------------------------------------

#### 进程共享属性

> `POSIX`中这个属性是可选的。
>
> 1. 可以在编译期间使用`_POSIX_THREAD_PROCESS_SHARED`符号来判断是否支持
> 2. 也可以在运行期间以`_SC_THREAD_PROCESS_SHARED`作为参数传递给`sysconf`函数来检测是否支持
>
> 在UNIX中有这样的机制：允许相互独立的多个进程把同一个内存数据块映射到它们各自独立的地址空间中。所以多个进程访问共享数据时，也需要同步。

下面是关于进程共享属性的操作函数

```c
#include <pthread.h>
int pthread_mutexattr_getpshared(const pthread_mutexattr_t *restrict attr,
                                int *restrict pshared);
int pthread_mutexattr_setpshared(pthread_mutexattr_t *attr,int pshared);
// 成功返回0，出错返回错误编号
```

* `pshared`为`PTHREAD_PROCESS_PRIVATE`时，允许线程库提供更有效的互斥量实现，这在多线程应用程序中是默认的情况。在多个**进程**共享多个互斥量的情况下，线程库可以限制开销较大的互斥量实现。
* `pshared`为`PTHREAD_PROCESS_SHARED`时，从多个进程彼此之间共享的内存数据块中分配的互斥量就可以用于这些**进程的同步**

------------------------------------------

#### 健壮属性

> 健壮属性与在多个进程间共享的互斥量有关。当持有互斥量的进程终止时，需要解决互斥量状态恢复的问题。这种情况发生时，互斥量处于锁定状态，恢复起来很困难。其他阻塞在这个锁的进程将会一直阻塞下去

下面是关于互斥量健壮属性的操作函数

```c
#include <pthread.h>
int pthread_mutexattr_getrobust(const pthread_mutexattr_t *restrict attr,
                               int *restrict robust);
int pthread_mutexattr_setrobust(pthread_mutexattr_t *attr,int robust);
// 成功返回0，出错返回错误编号
```

* 当`robust`为`PTHREAD_MUTEX_STALLED`时，意味着持有互斥量的进程终止时不需要采取特别的动作。这种情况下，使用互斥量后的行为是未定义的，等待该互斥量解锁的应用程序将有效的被拖住
* 当`robust`为`PTHREAD_MUTEX_ROBUST`时，会导致当一个线程调用`pthread_mutex_lock`获取锁时，而该锁被另一个进程持有，但它终止时并没有对该锁进行解锁，此时线程会阻塞，从`pthread_mutex_lock`返回的值为`EOWNERDEAD`而不是0。应用程序可以通过这个特殊值获知，若有可能，不管它们保护的互斥量状态如何，都需要进行恢复。

如果应用状态无法恢复，在线程对互斥量解锁后，该互斥量将处于永久不可用状态。可以使用下面的函数指明该互斥量相关的状态与互斥量解锁之前是一致的。

```c
#include <pthread.h>
int pthread_mutex_consistent(phtead_mutex_t *mutex);
// 成功返回0，出错返回错误编号
```

> 如果线程没有先调用该函数就对互斥量进行了解锁，那么其他试图获取该互斥量的阻塞线程就会得到错误码`ENOTRECOVERABLE`。如果发生这种情况，互斥量将不再可用
>
> 所以一定要在解锁之前调用该函数，才能使互斥量正常工作。

--------------------------------

#### 类型属性

> 类型互斥量属性控制着互斥量的锁定特性
>
> `PTHREAD_MUTEX_NORMAL`：标准互斥量类型，不做任何特殊的错误检查和死锁检测
>
> `PTHREAD_MUTEX_ERRORCHECK`：提供错误检查的互斥量类型
>
> `PTHREAD_MUTEX_RECURSICE`：允许同一线程在互斥量解锁之前对该互斥量进行多次加锁。递归互斥量会维护锁的计数，在解锁次数和加锁次数不同的情况下，不会释放锁
>
> `PTHREAD_MUTEX_DEFAULT`：提供默认特性和行为的互斥量类型。操作系统会把它映射到上面三种类型之一

下面关于类型属性操作的函数：

```c
#include <pthread.h>
int pthread_mutexattr_gettype(const pthread_mutexattr_t *restrict attr,
                             int *restrict type);
int pthread_mutexattr_settype(pthread_mutexattr_t *attr,int type);
```

### 读写锁属性

初始化和销毁读写锁属性的函数

```c
#include <pthread.h>
int pthread_rwlockattr_init(pthread_rwlockattr_t *attr);
int pthread_rwlockattr_destroy(pthread_rwlockattr_t *attr);
// 成功返回0，出错返回错误编号
```

读写锁的唯一属性就是`进程共享属性`。它和互斥量的`进程共享属性`是相同的。

操作该属性的函数

```c
#include <pthread.h>
int pthread_rwlockattr_getpshared(const pthread_rwlockattr_t *restrict attr,
                                 int *restrict pshared);
int pthread_rwlockattr_setpshared(pthread_rwlockattr_t *attr,int pshared);
// 成功返回0，出错返回错误编号
```

* 虽然只定义了一个读写锁属性，但是不同平台的实现可以自由的定义额外的、非标准的属性

### 条件变量属性

初始化和销毁条件属性的函数

```c
#include <pthread.h>
int pthread_condattr_init(pthread_condattr_t *attr);
int pthread_condattr_destroy(pthread_condattr_t *attr);
// 成功返回0，出错返回错误编号
```

* 条件变量有两个属性
  * 进程共享属性：与其他同步属性一样。
  * 时钟属性：控制`pthread_cond_timedwait`函数的超时参数`tsptr`时采用的是哪个时钟

操作这两个属性的函数如下：

```c
#include <pthread.h>
int pthread_condattr_getpshared(const pthread_condattr_t *restrict attr,
                               int *restrict pshared);
int pthread_condattr_setpshared(pthread_condattr_t *attr,int pshared);

int pthread_condattr_getclock(const pthread_condattr_t *restrict attr,
                             clockid_t *restrict clock_id);
int pthread_condattr_setclock(pthread_condattr_t *attr,clockid_t clock_id);
// 成功返回0，出错返回错误编号

// clockid_t的取值如下
/*
CLOCK_REALTIME
CLOCK_MONOTONIC
CLOCK_PROCESS_CPUTIME_ID
CLOCK_THREAD_CPUTIME_ID
含义在--系统数据文件和信息--中有记录
*/
```

### 屏障属性

初始化和销毁函数

```c
#include <pthread.h>
int pthread_barrierattr_init(pthread_barrierattr_t *attr);
int pthread_baarierattr_destroy(pthread_barrierattr_t *attr);
// 成功返回0，否则返回错误编号
```

只有一个属性：进程共享属性。处理函数如下

```c
#include <pthread.h>
int pthread_barrierattr_getpshared(const pthread_barrierattr_t *restrict attr,
                                  int *restrict pshared);
int pthread_barrierattr_setpsahred(pthread_barrierattr_t *attr,int pshare);
// 成功返回0，出错返回错误编号
```

## 重入

如果一个函数在相同的时间点可以被多个线程安全的调用，就称该函数是线程安全的。

如果函数对异步处理程序的重入是安全的，那么就可以说函数是异步信号安全的。

很多函数在多线程调用时不安全主要是因为它们使用了静态存储区或其他共享存储等。

`POSIX`提供了以线程安全的方式管理`FILE`对象的方法。

```c
#include <stdio.h>
int ftrylockfile(FILE *fp);
// 成功返回0，失败返回错误码（非0值）
void flockfile(FILE *fP);
void funlockfile(FILE *fp);
```

* 这个锁是递归的：当你占有这把锁时，还可以再次获取这把锁，而且不会导致死锁

如果使用的每次一个字符的IO时，每次都进行加锁解锁会非常的消耗性能。为了避免这种加解锁的开销，可以使用下面这四个函数

```c
#include <stdio.h>
int getchar_unlocked(void);
int getc_unlocked(FILE *fp);
// 成功解放者下一个字符，遇到文件尾或失败返回EOF
int putchar_unlocked(int c);
int putc_unlocked(int c,FILE *fp);
// 成功返回c，出错返回EOF
```

* 除非被`flockfile`（或`ftrylockfile`）和`funlockfile`的调用包围，否则尽量不要调用这四个函数，因为他们会导致不可预期的结果。
* 一旦对`FILE`对象进行加锁，就可以在释放锁之前对这些函数进行多次调用。这样就可以在多次的数据读写上分摊总的加解锁的开销

## 线程特定数据

> 线程特定数据，也称为线程私有数据，是存储和查询某个特定线程相关数据的一种机制。
>
> 提供这种机制的原因有两个：
>
> 1. 有时候需要维护基于每个线程的数据，防止某个线程的数据与其他线程的数据相混淆。
> 2. 这个机制提供了让基于进程的接口适应多线程环境的机制。例如`errno`

在分配线程特定数据之前，需要创建与该数据关联的键。这个键用于获取对线程特定数据的访问。

```c
#include <pthread.h>
int pthread_key_create(pthread_key_t *keyp,void (*destructor)(void *));
// 成功返回0，出错返回错误编号
```

* 创建的key存储在`keyp`指向的内存单元中
* 这个键可以被进程中的所有线程使用，但每个线程把这个键与不同的线程特定数据地址进行关联。
* 创建新键时，每个线程的数据地址设为空值
* 第二个参数为是一个析构函数的地址。只有当线程正常退出时才会调用该析构函数。异常终止是不会调用的

我们可以用下面的函数来取消键与线程特定数据之间的关联

```c
#include <pthread.h>
int pthread_key_delete(pthread_key_t *key);
// 成功返回0，出错返回错误编号
```

* 调用该函数不会激活与键有关的析构函数

解决多个线程多次调用创建键的函数，可以使用下面这个函数

```c
#include <pthread.h>
pthread_once_t initflag = PTHREAD_ONCE_INIT;
int pthread_once(pthread_once_t *initflag,void (*initfn)(void));
// 成功返回0，出错返回错误编号
```

* `initflag`必须是一个非本地变量(如全局变量或静态变量)，而且必须初始化为`PTHREAD_ONCE_INIT`
* 每个线程都可以调用该函数，但是该函数能保证初始化例程`initflag`只被调用一次

键一旦创建以后，就可以使用下面的函数与线程特定数据进行关联，并可以通过键获取特定数据

```c
#include <pthread.h>
void *pthread_getspecific(pthread_key_t key);
// 返回线程特定数据值，没有返回NULL
int pthread_setspecific(pthread_key_t key,const void *value);
// 成功返回0，出错返回错误编号
```

## 取消选项

> 可取消状态和可取消类型这两个属性影响着线程在响应`pthread_cancel`函数调用时所呈现的行为。

设置可取消状态的函数

```c
#include <pthread.h>
int pthread_setcancelstate(int state, int *oldstate);
// 成功返回0，出错返回错误码
```

* state参数的取值
  * `PTHREAD_CANCEL_ENABLE`
  * `PTHREAD_CANCEL_DISABLE`
* 通过`oldstate`返回原来的状态

`pthread_cancel`并不等待线程终止。在默认情况下，线程在取消请求发出后还是继续运行，知道线程到达某个取消点。取消点是线程检测它是否被取消的一个位置，如果被取消了，则按照要求行事。取消点的函数请查阅书籍

当把状态设置为`PTHREAD_CANCEL_DISABLE`时，`pthread_cancel`调用并不会杀死线程。相反，取消请求对这个线程来说还处于挂起状态，当取消状态变为`PTHREAD_CANCEL_ENABLE`时，线程的将会在下次一个取消点对所挂起的取消请求进行处理

如果线程在很长一段时间内不调用带有取消点的函数，可以手动的调用下面这个函数进行检测

```c
#include <pthread.h>
void pthread_testcancel(void);
```

前面所描述的取消类型被称为延迟取消。也就是说在调用取消请求后，在线程到达取消点之前，线程都不会真正的被取消。可以通过下面这个函数来修改取消类型

```c
#include <pthread.h>
int pthread_setcanceltype(int type, int *oldtype);
// 成功返回0，出错返回错误码
```

* type的取值
  * `PTHREADCANCEL_DEFERRED`——推迟取消
  * `PTHREADCANCEL_ASYNCHRONOUS`——异步取消。线程可以在任意时间撤销，不是非要遇到取消点菜被取消

## 线程与信号

> 每个线程都有自己的信号屏蔽字，但是信号的处理是进程中所有线程共享的。这就意味着当一个线程阻塞某个信号，但是当另一个线程修改了与这个信号相关的处理行为后，所有的线程都会共享这个处理行为的改变。
>
> 进程中的信号是递送到单个线程的。如果一个信号与硬件故障相关，那么该信号一般会被发送引起该事件的线程中去，而其他类型的信号则被发送到任意一个线程。

进程中使用`sigprocmask`函数来处理信号屏蔽字的，在线程中使用下面的函数来处理：

```c
#include <pthread.h>
int pthread_sigmask(int how,const sigset_t *restrict set,sigset_t *restrict oset);
// 成功返回0，出错返回错误编号
```

* 这个函数的参数与`sigprocmask`函数基本相同。
* 只是返回值不同，该函数失败会返回错误码，而不是返回-1，并设置`errno`值

线程可以通过调用下面的函数，来等待一个或多个信号的出现

```c
#include <signal.h>
int sigwait(const sigset_t *restrict set,int *restrict signop);
// 成功返回0，出错返回错误编号
```

* `set`参数指定了线程等待的信号集。
* 返回时，`signop`指向的整数将包含发送信号的数量。
* 如果信号集中的某个信号在函数调用的时候处于挂起状态，那么该函数将会无阻塞的返回。在返回之前，该函数将从进程中移除那些处于挂起等待状态的信号。
* 如果一个信号被捕获，而且一个线程正在`sigwait`调用中等待同一个信号，那么这时将由操作系统实现来决定以何种方式递送信号。
  * 可以让`sigwait`函数返回
  * 也可以激活信号处理程序
  * 但是上面这两种情况不会同时发生

发送指定信号给线程的函数

```c
#include <pthread.h>
int pthread_kill(pthread_t thread,int signo);
// 成功返回0，出错返回错误编号
```

* 与`kill`函数一样，可以用`0`值的`signo`来检查线程是否存在。
* 如果信号的默认处理动作是终止进程，那么用该函数发送信号给线程也会终止掉整个进程
* 注意，闹钟定时器是进程资源，并且所有的线程共享相同的闹钟。所以，在进程中的多个线程不可能互不干扰的使用闹钟定时器

## 线程与fork

当线程调用`fork`时，就为子进程创建了整个进程地址空间的副本。

子进程通过继承整个地址空间的副本，还从父进程那里继承了每个互斥量、读写锁和条件变量的状态。

如果父进程包含一个以上的线程，子进程在`fork` 返回以后，如果没有紧接着调用`exec` 的话，就需要**清理锁状态**

在子进程内部只有一个线程，它是由父进程中调用`fork`的线程的副本构成的。

如果父进程中的线程占有锁，子进程将同样占有这些锁。问题是子进程可能并不包含占有锁的线程的副本，所以子进程就不知道他占有那些锁，也不知道它需要清理那些锁。

`POSIX`声明，在`fork`返回和子进程调用其中一个`exec`函数之间，子进程只能调用异步信号安全的函数

要清理锁状态，可以通过下面这个函数建立`fork` 处理程序

```c
#inlcude <pthread.h>
it pthread_atfork(void (*prepare)(void), void (*parent)(void), void (*child)(void));
// 成功返回0，出错返回错误码
```

* `prepare`处理程序是父进程在创建子进程**之前**调用。这个处理程序可以用来**获取**父进程中定义的所有锁。
* `parent`处理程序是在创建子进程**之后**在**父进程**调用的。这个处理程序是用来处理对获取的所有锁进行解锁的
* `child`处理程序是在创建子进程**之后**在**子进程**中用的。这个处理程序也是用来处理对获取的所有锁进行解锁的
* 不会出现加一次锁，解锁两次的情况。因为创建子进程后，子进程就得到了父进程定义的所有锁的副本。
* 可以多次调用这个函数从而设置多套`fork`处理程序。如果不需要某个处理程序可以传入空指针
* 注册多个处理程序时，处理程序的调用顺序并不相同
  * `parent`和`child`处理程序是以他们注册的顺序进行调用的
  * `prepare`处理程序的调用顺序是与注册的顺序相反的
  * 这样可以允许多个模块注册自己的的`fork` 处理程序，而且保持锁的层次。