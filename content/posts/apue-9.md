---
title: 9.线程
date: 2020-07-30 11:30:14
draft: false
categories: 
- APUE
---

## 线程的概念

> 典型的UNIX进程可以看成只有一个控制线程：一个进程在某一时刻只能做一件事情。
>
> 有了多个控制线程后。可以有很多好处：
>
> * 通过为每种事件类型分配单独的处理线程，可以简化处理异步时间的代码。
> * 多个进程必须使用操作系统提供的复杂机制才能实现内存和文件描述符的共享。而多个线程自动的可以访问相同的存储地址空间和文件描述符
> * 有些问题可以分解从而提高整个程序的吞吐量
> * 交互的程序同样可以通过使用多线程来改善响应时间，多线程够可以把程序中处理用户输入和输出的部分与其他部分分开
>
> 每个线程都包含有表示执行环境所必须的信息，其中包括：
>
> * 线程ID
>* 一组寄存器值
> * 栈
> * 调度优先级
> * 策略
> * 信号屏蔽字
> * `errno`变量
> * 线程私有数据(后面会讲述)
> 
> 一个进程的所有信息对该进程的所有线程都是共享的，包括：
>
> * 可执行程序的代码
>* 程序的全局内存
> * 堆内存
> * 栈
> * 文件描述符
> 
> 后面记录的线程接口是`POSIX.1`的。接口名也叫`pthread`或`POSIX线程`

<!--more-->

## 线程标识

> 每个线程都有一个线程ID。
>
> 线程ID只有在它所属的进程上下文中才有意义

线程ID是用`pthread_t`数据类型来表示的。实现的时候可以用一个结构来代表`pthread_t`数据类型。所以可移植的操作系统实现不能把它作为整数处理。因此可以使用下面这个函数来比较两个线程ID。

```c
#include <pthread.h>
int pthread_equal(pthread_t tid1,pthread_t tid2);
// 相等返回非0，否则返回0
```

用`pthread_t`的后果是不能用一种可移植的方式打印该数据类型的值。

获取自身线程ID的函数

```c
#include <pthread.h>
pthread_t pthread_self(void);
// 返回调用线程的线程ID
```

## 线程创建

> 在传统的UNIX教程模型中，每个进程只有一个控制线程。这与基于线程的模型中每个进程只包含一个线程是相同的。

```c
#include <pthread.h>
int pthread_create(pthread_t *restrict tidp, 
                   const pthread_attr_t *restrict attr,
                   void *(*start_rtn)(void *),
                   void *restrict arg);
// 成功返回0，出错返回错误编码
```

* 当该函数成功返回时，新线程的线程ID会被设置为`tidp`指向的内存单元
* `attr`参数用于定制各种不同的线程属性。这个暂时设置为`NULL`，后面再详细讲解
* 新线程从`start_rtn`函数的地址开始运行。该函数只有一个无类型指针参数`arg`
* 如果需要向开始函数传递参数，需要把这些参数放到一个结构中，然后把结构的地址作为`arg`参数传入
* 新创建的线程可以访问进程的地址空间，并且继承调用线程的浮点环境和信号屏蔽字，但是该**线程的挂起信号集会被清除**
* 注意，该函数不像其他`POSIX`函数一样设置`errno`。每个线程都提供`errno`的副本，为了与使用`errno`的现有函数兼容，所以该函数返回出错编码

## 线程终止

> 如果在进程中的任一线程调用了`exit`、`_Exit`或者`_exit`，那么整个进程就会停止
>
> 如果默认的动作是终止进程，那么发送到线程的信号就会终止整个进程
>
> 单个线程可以通过3中方式退出：
>
> 1. 线程可以简单的从启动例程中返回，返回值是线程的退出码
> 2. 线程可以被同一进程中的其他线程取消
> 3. 线程可以调用`pthead_exit`
>
> 通过这三种方式，可以在不终止整个进程的情况下，停止它的控制流

```c
#include <pthread.h>
void pthread_exit(void *rval_ptr);
```

* `rval_ptr` 参数是一个无类型指针，与传给启动例程的单个参数类似
* 进程中的其他线程也可以调用`pthread_join`函数访问`rval_ptr`

```c
#include <pthread.h>
int pthread_join(pthread_t thread,void **rval_ptr);
// 成功返回0，出错返回错误编号
```

* 调用该函数的线程将一直阻塞，直到指定线程调用`pthread_exit`、从启动例程中返回或被取消才会结束阻塞。
* 如果线程简单的从它的启动例程返回，`rval_ptr`将包含返回码。
* 如果线程被取消，则由`rval_ptr`指定的内存单元就设置为`PTHREAD_CANCELED`
* 可以通过调用`pthread_join`自动把线程置于分离状态，这样资源就可以恢复
* 如果线程已经处于分离状态，`pthread_join`调用失败，返回`EINVAL`
* 如果对返回值不感兴趣，可以吧`rval_ptr`设置为`NULL`

----------------------------------------------

**注意：`pthread_exit`和`pthread_join`函数的无类型指针参数可以传递的值不值一个，这个指针可以传递包含复杂信息的结构的地址，但是注意，这个结构所使用的内存在调用者完成调用后必须任然是有效的**

原因是：同一进程内的所有线程，共享堆栈存储区。

-------------------------------------------------------

进程可以通过`pthread_cancel`函数来请求取消同一进程中的其他线程

```c
#include <pthread.h>
int pthread_cancel(pthread_t tid);
// 成功返回0，出错返回错误编号
```

* 默认情况下，该函数会使得有`tid`标识的线程的行为表现为如同调用了参数为`PTHREAD_CANCELED`的`pthread_exit`函数，但是线程可以选择忽略取消或者控制如何被取消
* `pthread_cancel`并不等待线程终止，它仅仅提出请求

线程退出时可以安排需要调用的处理函数，这与安排进程的退出处理函数`atexit`类似。这样的函数被称为`清理处理函数`。

```c
#include <pthread.h>
void pthread_cleanup_push(void (*rtn)(void *),void *arg);
void pthread_cleanup_pop(int execute);
```

* 处理程序记录再栈中，因此他们的调用顺序和注册顺序相反

* 清理函数`rtn`由`pthread_cleanup_push`函数调度，调用时只有一个参数`arg`

  * 调用`pthread_exit`时
  * 相应取消请求时；
  * 用非零`execute`参数调用`pthread_cleanup_pop`时

* 如果`execute`参数设置为0，清理函数将不被调用。不管发生什么上述3中情况中的那种，`pthread_cleaup_pop`都将删除上次`pthread_cleanup_push`调用建立的清理处理程序

* 这些函数有一个限制：由于它们可以实现为宏，所以必须在于线程相同的作用域中以匹配对的形式使用。例如：

  ```c
  void *thr_fn1(void *arg)
  {
      pthread_cleanup_push(cleanup,"thread 1 first handler");
      pthread_cleanup_push(cleanup,"thread 2 first handler");
      ...
      pthread_cleanup_pop(0);
      pthread_cleanup_pop(0);
  }
  ```

  * 如果线程是通过从它的启动例程中返回而终止的话，它的清理程序就不会被调用

  处理线程的函数和处理进程的函数很相似

  | 进程原语  |        线程原语        |             描述             |
  | :-------: | :--------------------: | :--------------------------: |
  |  `fork`   |    `pthread_create`    |        创建新的控制流        |
  |  `exit`   |     `pthread_exit`     |      从现有控制流中退出      |
  | `waitpid` |     `pthread_join`     |    从控制流中得到退出状态    |
  | `atexit`  | `pthread_cleanup_push` | 注册在退出控制流时调用的函数 |
  | `getpid`  |     `pthread_self`     |         获得控制流ID         |
  |  `abort`  |    `pthread_cancel`    |    请求控制流的非正常退出    |

------------------------------------------

>  在默认情况下，线程的终止状态会保存在直到对线程调用`pthread_join`。如果线程已经被分离，线程的底层存储资源可以在线程终止时立即被回收。在线程被分离后，我们不能用`pthread_join`函数等待它的终止状态，因为对分离状态的线程调用`pthread_join`会产生未定义的行为。

分离线程的函数

```c
#include <pthread.h>
int pthread_detach(pthread_t tid);
// 成功返回0，出错返回出错编号
```

## 线程同步

> 当多个控制线程共享相同内存时，需要确保每个线程看到一致的数据视图。
>
> 当一个线程可以修改的变量，其他线程也可以读取或者修改的时候，我们就需要对这些线程进行同步，确保它们在访问变量的存储内容时不会访问到无效的值
>
> 当一个线程修改变量时，其他线程在读取这个变量时可能会看到一个不一致的值。为了解决这个问题，可以使用线程锁，同一时间只允许一个线程访问该变量。

### 互斥量

> 可以使用`pthread`的互斥接口来保护数据，确保同一时间只有一个线程访问数据。
>
> `互斥量(mutex)`从本质上来说就是一把锁，在访问共享资源前对互斥量加锁，在访问完成后解锁互斥量。
>
> 只有将所有的线程都设计成遵守相同数据访问规则的，互斥机制才能正常工作。操作系统不会为我们做数据访问的串行化

* 互斥量是用`pthread_mutex_t`数据类型表示的。
* 在使用互斥量之前，必须向对它进行初始化。
* 如果是静态分配的互斥量，可以直接设定为`PTHREAD_MUTEX_INITIALIZER`，可以使用下面的init函数初始化 
* 如果是动态分配的互斥量(`malloc`)，在释放前要调用`pthread_mutex_destroy`

```c
#include <pthread.h>
int pthread_mutex_init(pthread_mutext_t *restrict mutex,
                       const pthread_mutexattr_t *restrict attr);
int pthread_mutex_destory(pthread_mutext_t *mutex);
// 成功返回0，出错返回错误编号
```

* 要是用默认属性的互斥量可以把`attr`设为`NULL`，`attr`参数在后面补充

对互斥量进行加锁和解锁的函数

```c
#include <pthread.h>
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutext_t *mutex);
// 成功返回0，出错返回错误编号
```

* 如果线程不希望被阻塞，它可以调用`pthread_mutex_trylock`尝试对互斥量进行加锁。
  * 如果调用时互斥量处于未锁住状态，那么该函数会锁住互斥量，不会出现阻塞直接返回0
  * 否则该调用就会失败，不能锁住互斥量，返回`EBUSY`

### 避免死锁

如果线程试图对同一个互斥量加锁两次，那么它自身就会陷入死锁状态。

可以通过仔细控制互斥量加锁的顺序来避免死锁的发生。

如果涉及了太多的锁和数据结构，可用的函数并不能把它转换成简单的层次，那么就需要采用另外的方法。此时，可以先释放占有的锁，然后过一段时间在试。这种情况下使用`trylock`加锁的方式是最好的

>多线程的软件设计需要代码复杂性和性能之间找到正确的平衡。如果锁的粒度太粗，就会出现很多线程阻塞等待相同的锁，这样并不能改善并发性。如果锁的粒度太细，那么过的锁开销会是系统效果受到影响，而且代码会非常的难以维护和阅读

### 函数`pthread_mutex_timedlock`

当线程试图获取一个已加锁的互斥量时，`pthread_mutex_timedlock`互斥量原语允许绑定线程阻塞时间。

```c
#include <pthread.h>
#include <time.h>
int pthread_mutex_timedlock(pthread_mutex_t *restrict mutex,
                           const struct timespec *restrict tsptr);
// 成功返回0，出错返回错误编号
```

* 参数`tsptr`是值的**绝对时间**
* 该函数等价于`pthread_mutex_lock`。但是在超过时间值时，该函数不会对互斥量进行加锁，而是出错返回，错误码为`ETIMEOUT`。

### 读写锁

> 读写锁与互斥量类似，不过读写锁有更高的并行性
>
> 读写锁有三种状态：
>
> * 读模式下加锁
> * 写模式下加锁
> * 不加锁
>
> 一次只有一个线程可以占有写模式的读写锁，但是多个线程可以同时占有读模式的读写锁
>
> 当读写锁是写加锁状态时，在这个锁被解锁前，所有试图对这个锁加锁的线程都会被阻塞
>
> 当读写锁处于读模式锁住的状态，而这是有一个线程试图以一个写模式获取锁时，读写锁通常会阻塞随后的度模式锁请求。这样就可以避免读模式锁长时间占用，而等待的写模式锁请求一直得不到满足
>
> 读写锁非常适合对于数据结构读的次数远大于写的情况

```c
#include <pthread.h>
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
                       const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);
// 成功返回0，出错返回错误编号
```

* 如果默认属性就足够的话，除了`init`函数也可以用`PTHREAD_RWLOCK_INITIALIZER`对静态分配的读写锁进行初始化
* 一定要在释放读写锁占用的内存之前调用`pthread_rwlock_destroy`做清理工作，不然分配的锁的资源会丢失

添加锁的方式

```c
#include <pthread.h>
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock); // 读的方式添加锁
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock); // 写的方式添加锁
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
// 成功返回0，出错返回错误编号
```

* 各种实现可能会对读模式下可获取的读写锁的次数进行限制，所有需要检测`pthread_rwlock_rdlock`的返回值

读写锁原语的条件版本。功能与互斥量的版本相同

```c
#include <pthread.h>
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);
// 成功返回0，出错返回错误编号
```

### 带有超时的读写锁

与互斥量的超时版本相同

```c
#include <pthread.h>
#include <time.h>
int pthread_rwlock_timedrdlock(pthread_rwlock_t *restrict rwlock,
                              const struct timespec *restrict rsptr);
int pthread_rwlock_timedwrlock(pthread_rwlock_t *restrict rwlock,
                              const struct timespec *restrict reptr);
// 成功返回0，出错返回错误编号
```

### 条件变量

> 条件变量是线程可用的另一种同步机制
>
> 条件变量给多个线程提供了一个会合的场所
>
> 条件变量与互斥量一起使用时，允许线程以无竞争的方式等待特定的条件发生
>
> 条件本身是由互斥量保护的。线程在改变条件之前必须首先锁住互斥量。其他线程在获得互斥量之前不会察觉到这种改变，因为互斥量必须在锁定以后才能计算条件。

```c
#include <pthread.h>
int pthread_cond_init(pthread_cond_t *restrict cond,
                     const pthread_condattr_t *restrict attr);
int pthread_cond_destroy(phtread_cond_t *cond);
// 成功返回0，出错返回错误编号
```

* 可以把常量`PTHREAD_COND_INITIALIZER`赋给静态分配的条件变量

```c
#include <pthread.h>
int pthread_cond_wait(pthread_cond_t *restrict cond,
                     pthread_mutex_t *restrict mutex);
int pthread_cond_timedwait(pthread_cond_t *restrict cond,
                           pthread_mutex_t *restrict mutex,
                          const struct timespec *restrict tsptr);
// 成功返回0，出错返回错误编号
```

* 使用`pthread_cond_wait`函数等待条件变量为真

传递给`pthread_cond_wait`的互斥量对条件进行保护。调用者把**锁住**的互斥量传给函数，函数然后自动把调用线程放到等待条件的线程列表上，**对互斥量解锁**。这就关闭了条件检查和线程进入休眠状态等待条件改变这两个操作之间的时间通道，这样线程就不会错过条件的任何变化。

`pthread_cond_wait`返回时，互斥量再次被锁住

两个函数可以用于通知线程条件已经满足。

```c
#include <pthread.h>
int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
// 成功返回0.出错返回错误编号
```

* `phtread_cond_signal`函数至少唤醒一个等待该条件的线程，
* `pthread_cond_broadcast`函数则能唤醒等待该条件的所有线程
* 注意：一定要在改变条件状态后在给线程发信号

### 自旋锁

> 自旋锁可用于以下情况：锁被持有的时间短，而且线程并不希望在重新调度上花费太多成本。
>
> 自旋锁通常作为底层原语用于实现其他类型的锁。
>
> 该锁会导致CPU资源浪费：当线程自旋等待锁变为可用时，CPU不能做其他的事情。这是自旋锁只能维持一段时间的原因
>
> 当自旋锁用在非抢占式内核中是非常有用的：除了提供互斥机制以外，他们会阻塞中断，这样中断处理程序就不会然该系统陷入死锁状态，因为他们需要获取已被加锁的自旋锁。

自旋锁的接口与互斥量的几口类似

```c
#include <pthread.h>
int pthread_spin_init(pthread_spinlock_t *lock,int pshared);
int pthread_spin_destroy(pthread_spinlock_t *lock);
int pthread_spin_lock(pthread_spinlock_t *lock);
int pthread_spin_trylock(pthread_spinlock_t *lock);
int pthread_spin_unlock(pthread_spinlock_t *lock);
// 成功返回0，否则返回错误编号
```

* 初始化函数多了一个`pshared`参数，它表示进程共享属性：
  * 如果设置为`PTHREAD_PROCESS_SHARED`，则自旋锁能被可以访问锁底层内存的线程所获取
  * 如果设置为`PTHREAD_PROCESS_PRIVATE`，自旋锁就只能被初始化该锁的进程内部线程锁访问
* 对已经锁住的自旋锁进行加锁，结果是未定义的，有些实现会返回错误码，有些实现会造成永久自旋
* 对没有锁住的自旋锁进行解锁，结果也是未定义的。
* 需要注意，不要调用在持有自旋锁情况下可能会进入休眠状态的函数。因为会浪费CPU资源，自旋的时间增加了

### 屏障

> 屏障是用户协调多个线程进行工作的同步机制。
>
> 屏障允许每个线程等待，直到所有的合作线程都达到某一点，然后从该点继续执行。

初始化和销毁函数

```c
#include <pthread.h>
int pthread_barrier_init(pthread_barrier_t *restrict barrier,
                        const pthread_barrierattr_t *restrict attr,
                        unsigned int count);
int pthread_barrier_destroy(pthread_barrier_t *barrier);
// 成功返回0，否则返回错误编码
```

* 初始化屏障是，使用`count`参数指定，在允许所有线程继续运行之前，必须到达屏障的线程数目
* `count`一经设置，就无法改变。可以使用销毁函数后，再次初始化

等待其他线程的函数

```c
#include <pthread.h>
int pthread_barrier_wait(pthread_barrier_t *barrier);
// 成功返回0或PTHREAD_BARRIER_SERIAL_THREAD，否则返回错误编号
```

* 当到达的线程数达到设定值时，所有的线程都会被唤醒
