---
title: emmc boot1 boot2 partition
date: 2016-11-14 23:29:25
tags: I.MX6
categories: I.MX6
---

使用mfg tool烧写android5.1的镜像之后，再使用旧版的mfg tool烧写linux或者android镜像，都不能正常启动，而且运行的uboot还是android5.1版本的uboot。

#### 参考链接

http://www.itdadao.com/articles/c15a39492p0.html

http://www.cnblogs.com/heiyue/p/5199851.html#undefined

http://www.cnblogs.com/heiyue/p/5830505.html

https://linux.codingbelief.com/zh/storage/flash_memory/emmc/emmc_partitions.html

https://community.nxp.com/thread/331306

https://freescale.jiveon.com/thread/429311

http://blog.5ibc.net/p/80599.html

#### emmc boot分区

在同事的帮助下才知道emmc有boot1，boot2以及RPMB(Replay Protected Memory Block),GPAP(General Purpose Area Partitions,最多可以有4个)，UDA（User Data Area)分区。而我们一般只知道UDA分区。

我们通常对emmc进行分区，也只是对UDA分区。boot1,boot2,RPMB分区是固定的，我们并不能控制器大小和分区格式。如下图所示（图片摘自网络）：

![](http://images2015.cnblogs.com/blog/745188/201611/745188-20161111105215624-775600562.png)


这次的android5.1镜像将uboot烧写到了boot1分区，并且将boot1分区使能。那么emmc每次启动之后就从boot1分区启动，运行android5.1的uboot，而android5.1的uboot又找不到其对应的kernel等镜像（因为UDA的区域的镜像已经烧写成别的版本），所以就一直卡在启动的页面。 

并且写到emmc寄存器中的值是一直保留的,不会因为开机就恢复为默认值。

而原来的工具是将uboot放在UDA分区。

android5.1的mfg tool中，ucl2.xml有那么一段。(我添加了注释)
```
<!-- 使emmc的boot0分区变为可读写 -->
	<CMD state="Updater" type="push" body="$ echo 0 > /sys/block/mmcblk%mmc%boot0/force_ro">access boot partition 1</CMD>
	<CMD state="Updater" type="push" body="send" file="files/android/%folder%/u-boot-imx%soc%%plus%.imx" >Sending u-boot.bin</CMD>
<!-- 将uboot烧录到emmc的boot0分区,在系统盘符里分区的序号是从0开始，也就是boot0, boot1 -->
	<CMD state="Updater" type="push" body="$ dd if=$FILE of=/dev/mmcblk%mmc%boot0 bs=512 seek=2">write U-Boot to sd card</CMD>
<!-- 使emmc的boot0分区变为只读 -->
	<CMD state="Updater" type="push" body="$ echo 1 > /sys/block/mmcblk%mmc%boot0/force_ro"> re-enable read-only access </CMD>
<!-- 将boot0分区使能，那么启动的时候就会从boot1分区启动,
     emmc寄存器序号是从1开始,boot1,boot2.这里的enable 1指的是emmc boot分区的第一个分区,是针对emmc的寄存器而言的。
     而上面的/sys/block/mmcblk3boot0指是针对linux系统而言的，其实指的都是统一个分区，就是emmc boot的最开始的分区。 
-->
	<CMD state="Updater" type="push" body="$ mmc bootpart enable 1 1 /dev/mmcblk%mmc%">enable boot partion 1 to boot</CMD>
```

#### emmc boot寄存器

下图是emmc寄存器设置。

不使能引导，将BOOT_PARTITION_ENABLE设置为0，

使能boot1，将BOOT_PARTITION_ENABLE设置为1，

使能boot2，将BOOT_PARTITION_ENABLE设置为2，

使能UDA，将BOOT_PARTITION_ENABLE设置为7.

![](http://images2015.cnblogs.com/blog/745188/201611/745188-20161111105235405-155711438.png)


#### 解决方法

在uboot中有emmc命令，用于设置emmc的寄存器。uboot版本不同，命令有些不一样。
```
          FSL 2009.08 U-boot       u-boot-fslc branch patches-2014.07
args      mmc bootpart             mmc partconf dev boot_ack boot_partition partition_access

example for change boot partition to device 2 partition 1:
            mmc bootpart 2 1        mmc partconf 2 1 1 1

example for change boot partition to device 2 partition 0:
            mmc bootpart 2 0        mmc partconf 2 1 0 1
```

具体操作如下：

```
# uboot中首先查看emmc的编号
Tony> mmc list
FSL_SDHC: 0
FSL_SDHC: 1
FSL_SDHC: 2 (eMMC)
# 确定emmc的序号是2
# 查看emmc命令
Tony> mmc
mmc - MMC sub system
Usage:
mmc info - display info of the current MMC device
mmc read addr blk# cnt
mmc write addr blk# cnt
mmc erase blk# cnt
mmc rescan
mmc part - lists available partition on current mmc device
mmc dev [dev] [part] - show or set current mmc device [partition]
mmc list - lists available devices
mmc hwpartition [args...] - does hardware partitioning
  arguments (sizes in 512-byte blocks):
    [user [enh start cnt] [wrrel {on|off}]] - sets user data area attributes
    [gp1|gp2|gp3|gp4 cnt [enh] [wrrel {on|off}]] - general purpose partition
    [check|set|complete] - mode, complete set partitioning completed
  WARNING: Partitioning is a write-once setting once it is set to complete.
  Power cycling is required to initialize partitions after set to complete.
mmc bootbus dev boot_bus_width reset_boot_bus_width boot_mode
 - Set the BOOT_BUS_WIDTH field of the specified device
mmc bootpart-resize <dev> <boot part size MB> <RPMB part size MB>
 - Change sizes of boot and RPMB partitions of specified device
mmc partconf dev boot_ack boot_partition partition_access
 - Change the bits of the PARTITION_CONFIG field of the specified device
mmc rst-function dev value
 - Change the RST_n_FUNCTION field of the specified device
   WARNING: This is a write-once field and 0 / 1 / 2 are the only valid values.
mmc setdsr <value> - set DSR register value

# 设置emmc的启动分区，主要是partconf后面的第一个和第三个参数。
# 2是emmc的编号，第三个参数设置启动的分区，对应寄存器BOOT_PARTITION_ENABLE字段。设为0表示不使能。 
Tony> mmc partconf 2 0 0 0
# 或者设置为7表示从UDA启动, 0和7我都尝试了，烧录原来的镜像都能够启动成功。
Tony> mmc partconf 2 0 7 0
```
或者更改ucl2.xml文件，使能UDA分区。
```
<!-- 但是设置成0时，烧录出错，不知道为什么 -->
<CMD state="Updater" type="push" body="$ mmc bootpart enable 0 1 /dev/mmcblk%mmc%">enable boot partion 1 to boot</CMD>
或者
<!-- 设置成7，烧录成功，再烧录原来的系统，能正常运行 -->
<CMD state="Updater" type="push" body="$ mmc bootpart enable 7 1 /dev/mmcblk%mmc%">enable boot partion 1 to boot</CMD>
```
#### Author

Tony Liu 

2016-11-11, Shenzhen
