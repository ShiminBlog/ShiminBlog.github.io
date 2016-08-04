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

### 文件存入与读取

怎么放入 splash 分区，直接影响到怎么从 splash 分区读取文件。客户指定的 splash 分区一共是 20M，经过实际测量和估算，bootloader 开机 logo 大约占 2M，如果 kernel logo 也做的话，大概也在 2M 左右。而一般来说，bootanimation.zip 和 shutdownanimation.zip 也都大概在 7M 左右。那么总的算下来， 2x2M + 7x2M = 18M。那么怎么存呢？前面说过，怎么存将影响到怎么取。无非就两种取法：

+ 系统在约定的启动阶段，去约定的 splash 地址，读取约定大小的文件(毕竟存入的是二进制文件)
+ 在 splash 分区特定的位置，放置一张表，表里说明，某个文件存放的位置，以及该文件的大小

对比两种取法，后面一种比前面一种灵活性更好，前面一种写法是在“取”的代码中把什么都写死了，一旦后面需要增加文件的大小，又得来修改这部分“取”的代码。而第二种方法则避免了这种缺陷。那这样吧，就将三个文件存放的起始地址、三个文件的大小，做成一张表，放在 splash 分区的最前面，每次读取文件的时候，就先检索该文件，然后根据该文件里面的值进行判定。为了更加准确，还需要加上一些头和尾，来作为校验。那么整个 splash 分区划定后如下：

![](/assets/splash/memory.png)

如上图所示，对整个splash分区进行了划定。下面来说明每个子分区里面的详细内容。

#### splash head

在 splash head 里面需要放置以下内容:
+ splash head 头标志，我这里使用字符串 "SPLASH!!" 来表示；
+ bootloader logo.png 的存放起始地址；
+ kernel logo.png 的存放起始地址；
+ bootanimation.zip 的存放起始地址；
+ shutdownanimation.zip 的存放起始地址

在上面的参数中，bootloader logo.png 、kernel logo.png 和 bootanimation.zip 的起始地址都是固定的，而 shutdownanimation.zip 的起始地址，是跟随者 bootanimation.zip 的起始地址加上其文件大小得出来的----这样做的目的是，将shutdownanimation.zip 接着 bootanimation.zip 挨着放，尽可能的节约空间，无他，仅此而已。另外，在代码中，因为当时为了验证的原因，多存入了 bootanimation.zip 和 shutdownanimation.zip 的大小参数，其实这俩参数也可以不要。

#### bootloader logo.png 和 kernel logo.png 区域

这里没有什么特别的，直接在指定的地址上，写入 logo.png 即可。不过代码中并没有真正的去存入和读取 kernel logo.png---因为kernel logo显示被我们关闭了。

#### bootanimation.zip 区域

在 bootanimation.zip 区域，分为了 32 bytes 的头、bootanimation.zip 文件以及32 bytes 的尾三部分。

其详细放置情况如下：
+ bootanimation.zip head 标志，我这里使用了 "BOOTANIMATION!!"；
+ 紧跟着 32 bytes 后放入 bootanimation.zip 文件大小的参数，以 byte 为单位，该参数占4 bytes.
+ 接着放入 bootanimation.zip 的二进制文件；
+ 接着放入 32 bytes 的尾部，在尾部的前 16 bytes，放入 bootanimation.zip 区域的结束标志，我这里使用了 "BOOTANIMATIONEND"。

那么，在开机动画阶段，读取 bootanimation.zip 的流程即是：
1. 读取 512 bytes 的 splash head 校验，并取出其中 bootanimation.zip 的首地址；
2. seek 到 bootanimation.zip 区域的首地址，读取 32 bytes 的 head，校验并取出其中该文件大小的参数；
3. 继续偏移 32 bytes，然后读取上述文件大小的字节数，放入缓存区；
4. 继续读取 32 bytes 的尾，并取出其中的 16 bytes 的尾并校验之；
5. 如果校验通过，则将第3步读取的数据写入到磁盘，结束读取 bootanimation.zip 文件的读取，否则，则返回失败。

#### shutdownanimation.zip 区域

在 shutdownanimation.zip 区域，分为了 32 bytes 的头、shutdownanimation.zip 文件以及32 bytes 的尾三部分。

其详细放置情况如下：
+ shutdownanimation.zip head 标志，我这里使用了 "SHUTDOWNANIMATION!!"；
+ 紧跟着 32 bytes 后放入 shutdownanimation.zip 文件大小的参数，以 byte 为单位，该参数占4 bytes.
+ 接着放入 shutdownanimation.zip 的二进制文件；
+ 接着放入 32 bytes 的尾部，在尾部的前 19 bytes，放入 shutdownanimation.zip 区域的结束标志，我这里使用了 "SHUTDOWNANIMATIONEND"。

那么，在关机动画阶段，读取 shutdownanimation.zip 的流程即是：
1. 读取 512 bytes 的 splash head 校验，并取出其中 shutdownanimation.zip 的首地址；
2. seek 到 shutdownanimation.zip 区域的首地址，读取 32 bytes 的 head，校验并取出其中该文件大小的参数；
3. 继续偏移 32 bytes，然后读取上述文件大小的字节数，放入缓存区；
4. 继续读取 32 bytes 的尾，并取出其中的 19 bytes 的尾并校验之；
5. 如果校验通过，则将第3步读取的数据写入到磁盘，结束读取 shutdownanimation.zip 文件的读取，否则，则返回失败。

那么，上述几个区域，形成的数据结构大致如下:

![](/assets/splash/struct.png)

其放置到 splash 分区的大概过程如下:

       boot_size, shut_size = CheckAnimation() 
	   print boot_size, shut_size
	   shutdownanimation_offset = bootanimation_offset + boot_size + 2*32 + 1
				
	   file = open(out, "wb")
	   file.write(MakeSplashIndex(boot_size, shut_size))
	   file.write(GetImgHeader(img.size))
	   file.write(GetImageBody(img))
		
	   file.seek(bootanimation_offset)       ## seek bootanimation.zip potision 

	   file.write(BootAnimationHead(boot_size))
	   file.write(BootAnimationBody())
	   file.write(BootAnimationEnd())

	   file.seek(shutdownanimation_offset)       ## seek bootanimation.zip potision
	   file.write(ShutdownAnimationHead(shut_size))
	   file.write(ShutdownAnimationBody())
	   file.write(ShutdownAnimationEnd())
	   file.close()


### bootloader 读取

这个可以在加载开机 logo 的代码之前去 emmc/flash的splash分区进行读取，此处不述。

### 开关机读取

在高通平台上，开关机动画读取流程都是放在以下文件的：

    frameworks/base/cmds/bootanimation/BootAnimation.cpp

在该文件的中 `readyToRun()` 里面，根据启动状态来判定当前是该调用开机动画还是关机动画: 

    int state = checkBootState() ? 0 : 1;

在这里，我们可以干掉原来的处理方法，根据 state 的值，来调用我们的读取函数，从splash分区读取开机动画或者关机动画。

**注意！！!**

这里需要注意的是，如果开关机动画无法成功读取，则开关机过程中，将会以 Android 原生图片进行显示: "ANDROID"。

## 难点处理

前面有提到在该次的实现过程中，有两处难点，其一就是权限问题，其二就是，字节对齐的问题。字节对齐很好处理，使用宏 `#pragma pack` 可以解决。这里给出解决方法以及所需权限结果。

如果我们在 .cpp 文件中，open/read/write 等操作时，没有权限，系统会提示，这些信息可以通过 logcat 或者 /proc/kmsg 下显示出来的。我这里解决的时候，是采用了kernel log 的方法，如下：

    cat /proc/kmsg | grep avc

以下文字为引用别人的blog：

> 遇到权限问题，在logcat或者kernel的log中一定会打印avc denied提示缺少什么权限，
 Command： 
  cat /proc/kmsg | grep avc 或 dmesg | grep avc 
  解决原则是：缺什么补什么，一步一步补到没有avc denied为止。  
  下面给出四个案例：  
  1. 
   audit(0.0:67): avc: denied { write } for path="/dev/block/vold/93:96" dev="tmpfs" ino=1263 scontext=u:r:kernel:s0 tcontext=u:object_r:block_device:s0 tclass=blk_file permissive=0  

   分析过程： 
   缺少什么权限：           { write }权限， 
   谁缺少权限：               scontext=u:r:kernel:s0， 
   对哪个文件缺少权限：tcontext=u:object_r:block_device 
   什么类型的文件：        tclass=blk_file  

   解决方法：kernel.te 
   allow kernel block_device:blk_file write;

以上详情请参看:  [Android 5.x 权限问题解决方法](http://m.blog.csdn.net/article/details?id=50904061)

当然，也不一定就需要把所有的都搞完。比如我这里，就只解决了 avc 出现的过程中，出现了 "bootani" 的部分内容。

以下是读取开关机动画所需要的权限:

*bootanim.te*

    allow bootanim block_device:dir search;
    allow bootanim mmc_block_device:blk_file rw_file_perms;
    allow bootanim system_data_file:dir {rw_file_perms search create add_name remove_name};
    allow bootanim system_data_file:file {rw_file_perms create setattr unlink};

*file_contexts*

    /dev/block/mmcblk0p29    u:object_r:mmc_block_device:s0

*ueventd.rc*

    /dev/block/mmcblk0p29     0777   root       graphics

*qseecomd.te*

    allow tee system_prop:property_service {set};

## 加入系统编译

让系统在 make 的时候，自动生成 splash.img，这是很重要的。这里以高通为例，修改如下:

*vendor/qcom/build/tasks/generate_extra_images.mk*

    INSTALLED_SPLASHIMAGE_TARGET := $(PRODUCT_OUT)/splash.img
    //....

![](/assets/splash/mk.png)

## 测试

测试该功能对系统的影响是否正常，可以通过创建一个空的 splash.img 或者一个内容不符合前面规格的 splash.img，烧入到 splash分区。看系统能否正常起来，显示的是否是android原生的"ANDOIRD"图片。然后再烧回正常的 splash.img，看能否正常显示即可。


2016-08-04

