---
title: 11.守护进程
date: 2020-07-30 11:30:23
draft: false
categories: 
- APUE
---

> 守护进程是一种生存期长的一种进程。
>
> 这里介绍守护进程结构，以及如何编写守护进程程序。本以为守护进程没有控制终端，我们需要了解在出现问题时，守护进程如何报告出错情况

<!--more-->

## 守护进程的特征

1. 大多数守护进程都是以超级用户特权运行的
2. 所有的守护进程都没有控制终端，其终端名设置为问号
3. 内核守护进程以无控制终端法师启动
4. 用户层守护进程缺少控制终端可能是守护进程调用了`setsid`的结果
5. 大多数用户层守护进程都是进程组的组长以及会话的首进程，而且是这些进程组和会话中的唯一进程
6. 用户层的守护进程的父进程都是`init`进程

## 编程规则

在编写守护进程 程序时需要遵循一些基础规则，以防止产生不必要的交互作用：

1. 首先要做的是调用`umask`将文件模式创建屏蔽字设置为一个已知值。
   1. 由继承得来的文件模式创建屏蔽字可能会这设置为拒绝某些权限
2. 调用`fork`，然后是父进程`exit`。这样做实现了下面几点
   1. 如果守护进程是作为一条简单的`shell`命令启动的，那么父进程会终止让`shell`认为这条命令已经执行完毕
   2. 虽然子进程继承了父进程的进程组ID，但获得了一个新的进程ID，这就保证了子进程不是一个进程组的组长进程
3. 调用`setsid`创建一个新的会话。这会使得调用进程有下面三个特性：
   1. 成为新会话的首进程
   2. 成为一个新进程组的组长进程
   3. 没有控制终端
4. 将工作目录更改为根目录。
   1. 从父进程那里继承过来的当前工作目录可能是在一个挂载的文件系统中。因为守护进程通常在系统再引导之前是一直存在的，所以如果守护进程的工作目录在一个挂载的文件系统中，那么该文件系统就不能被卸载
   2. 或者将工作目录修改到一个指定的位置
5. 关闭所有不再需要的文件描述符
   1. 可以使用`getrlimit`函数来判断最好文件描述符值，然后循环关闭所有文件描述符
6. 某些守护进程打开`/deb/null`使其具有文件描述符0、1和2，这样任何一个试图读标准输入、写标准输出和标准错误的库例程都不会产生任何效果

## 守护进程的惯例

1. 若守护进程使用锁文件，那么该文件通常存储在`/var/run`目录中
2. 若守护进程支持配置选项，那么配置文件通常存放在`/etc`目录中
3. 守护进程可用命令行启动，但通常它们是由系统初始化脚本之一（`/etc/rc*`或`/etc/init.d/*`）启动的
4. 若一个守护进程有一个配置文件，那么当该守护进程启动时会读取该文件，但在此后一般不会再查看它。
   1. 如果对配置文件进行了修改，那么该守护进程可能需要重新启动来加载配置文件。也可以捕捉`SIGHUP`信号来处理