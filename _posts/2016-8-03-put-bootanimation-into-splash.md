---
layout: post
title:  Put android bootanimation and shutdownanimation into splash zone
categories: 工作随笔
date: 2016-08-03 19:40:30
---

*** 

应一个客户的需要，需要在 Android 5.1 上，将 Android 的开机 logo、开关机动画 bootanimation.zip shutdownanimation.zip 一起存放在 splash 分区。下面就记述下实现的难点、思路以及大体逻辑。


## 难点

在整个实现的过程中，难点有两部分：

+ Android 5.1 权限问题，也就是 SElinux 问题；
+ 数据结构字节对齐问题

在后面的步骤中一一叙述之。


## 思路

将开机logo、kernel logo(如果有的话)、bootanimation.zip、shutdownanimation.zip 分别以二进制的形式存放在 splash 分区。为了系统能准确读出这几种文件，还需要将这几种文件的大小、以及存在 splash 分区的哪个位置，这两类参数也得保存到 splash 某个特定的位置。开机后，bootloader会首先取出开机logo进行显示，到了开机动画显示阶段，在显示之前，从 splash 分区取出 bootanimation.zip 然后显示，关机动画也是同样的道理。


## 实现

以下是实现的过程。

### 文件怎么放入 splash 分区

怎么放入 splash 分区，直接影响到怎么从 splash 分区读取文件。客户指定的 splash 分区一共是 20M，经过实际测量和估算，bootloader 开机 logo 大约占 2M，如果 kernel logo 也做的话，大概也在 2M 左右。而一般来说，bootanimation.zip 和 shutdownanimation.zip 也都大概在 7M 左右。那么总的算下来， 2x2M + 7x2M = 18M。那么怎么存呢？前面说过，怎么存将影响到怎么取。无非就两种取法：

+ 系统在约定的启动阶段，去约定的 splash 地址，读取约定大小的文件(毕竟存入的是二进制文件)
+ 在 splash 分区特定的位置，放置一张表，表里说明，某个文件存放的位置，以及该文件的大小

对比两种取法，后面一种比前面一种灵活性更好，前面一种写法是在“取”的代码中把什么都写死了，一旦后面需要增加文件的大小，又得来修改这部分“取”的代码。而第二种方法则避免了这种缺陷。那这样吧，就将三个文件存放的起始地址、三个文件的大小，做成一张表，放在 splash 分区的最前面，每次读取文件的时候，就先检索该文件，然后根据该文件里面的值进行判定。为了更加准确，还需要加上一些头和尾，来作为校验。那么整个 splash 分区划定后如下：

![](/assets/splash/memory.png)


