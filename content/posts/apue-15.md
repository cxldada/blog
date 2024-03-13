---
title: 15.终端IO
date: 2021-07-07 15:21:38
draft: false
categories: 
- APUE
---

终端IO总结

<!--more-->

## 综述

终端IO有两种不同的工作模式：
1. 规范模式输入处理。终端输入以行为单位。对每个读请求，终端驱动程序最多返回一行。这也是终端的默认工作模式
2. 非规范模式输入处理。输入字符不装配成行

所有可以检测和更改的终端设备特性都包含在`termios`结构中。

```c
#include <termios.h>

struct termios {
    tcflag_t c_iflag; // input flags
    tcflag_t c_oflag; // output flags
    tcflag_t c_cflag; // control flags
    tcflag_t c_lflag; // local flags
    cc_t c_cc[NCCS];  // control characters
};
```

