---
title: "Unpack and Repack Initramfs"
date: 2021-03-21T14:16:16+08:00
draft: false
tags: []
summary: ""
---

initramfs是一种嵌入在操作系统内核中的根文件系统(root filesystem),在boot的早期阶段就会载入,里面包含了完全启动系统所需的工具和文件.initramfs在完全启动后实际上并不会从内存中消失,这个[文件系统](https://www.linuxquestions.org/questions/linux-general-1/accessing-initrd-in-memory-after-boot-4175614932/?__cf_chl_jschl_tk__=3c085e1f32f31adec75525faf41b83178d0570bf-1616311609-0-AZj1LlCJkO4bsxNQm6gOccAtGIycIS-KJciOF3RCF59fzEWG7Su66dIdVpcN6QhWDBVODWFtm6aqPle4SIiL2JwLkR8uI-lvknKzG96qyNR75hCNjjugSDsnqL-qY4KpE5KIIT5cmsvL4sd3fr3D1DdL5gpux9MIhwPKKDBxfH_eS5ORM2PP_WfjZabU7fx_A1hjkeAvEhRJSrtF_vkTMDnd3a0QiGT1oPZUlcsVBx0bh1iE8jqgLjx61EojkBLuuZbnd_vXNBp5xHY_XmhBh02jZbQBEsvH2-1UvKg6r9R6BclHuql1lQuMoIS6NQLm4PPj45Bz0FBh1B6qRr-lP-JmMCmjBbOHLSamXaM8MjMATJC-ZlgpCr-iTc4YhRQ7_9eHFekwVue-j2RKoDjqbY8mHqKlbskrfBrsoZoxO-gsgoksBXK9BG0_McLpJykGyHsxD5izmOn1Hk_v5cmYZmeISBxrTAy5C7xvMt2RHDUr)位于/dev/initrd之类的设备下,虽然并没有找到.

本文将拆解initrd/initramfs boot image,并粗略分析操作系统挂载内核后的启动过程(正确性存疑,看看就行).

### Unpack initramfs

我们以Archlinux boot目录下的initramfs-linux.img为例解包.
分析initramfs文件类型:
``` shell
$ file /boot/initramfs-linux
/boot/initramfs-linux.img: Zstandard compressed data (v0.8+), Dictionary ID: None
```
至此我们知道Arch的initramfs文件是一个zst压缩包，下一步很简单解压这个文件就可以:
``` shell
$ cp /boot/initramfs-linux.img ./initramfs-linux.img.zst && zstd -d ./initramfs-linux.img.zst
$ ls -lh
-rwxr-xr-x  1 <username> <group> 11M Mar 21 01:40 initramfs-linux.img.zst
-rwxr-xr-x  1 <username> <group> 32M Mar 21 01:40 initramfs-linux.img
```
我们这样就得到一个32MB的文件,也就是initramfs的本体,这个文件内的内容使用cpio提取:
``` shell
$ cpio --extract --make-directories --format=newc --no-absolute-filenames < initramfs-linux.img
$ ls -lh
lrwxrwxrwx 1 <username> <group>    7 Mar 21 01:42 bin -> usr/bin
-rw-r--r-- 1 <username> <group> 2.5K Mar 21 01:42 buildconfig
-rw-r--r-- 1 <username> <group>   79 Mar 21 01:42 config
drwxr-xr-x 2 <username> <group> 4.0K Mar 21 01:42 dev
drwxr-xr-x 3 <username> <group> 4.0K Mar 21 01:42 etc
drwxr-xr-x 2 <username> <group> 4.0K Mar 21 01:42 hooks
-rwxr-xr-x 1 <username> <group> 2.1K Mar 21 01:42 init
-rw-r--r-- 1 <username> <group>  13K Mar 21 01:42 init_functions
lrwxrwxrwx 1 <username> <group>    7 Mar 21 01:42 lib -> usr/lib
lrwxrwxrwx 1 <username> <group>    7 Mar 21 01:42 lib64 -> usr/lib
drwxr-xr-x 2 <username> <group> 4.0K Mar 21 01:42 new_root
drwxr-xr-x 2 <username> <group> 4.0K Mar 21 01:42 proc
drwxr-xr-x 2 <username> <group> 4.0K Mar 21 01:42 run
lrwxrwxrwx 1 <username> <group>    7 Mar 21 01:42 sbin -> usr/bin
drwxr-xr-x 2 <username> <group> 4.0K Mar 21 01:42 sys
drwxr-xr-x 2 <username> <group> 4.0K Mar 21 01:42 tmp
drwxr-xr-x 5 <username> <group> 4.0K Mar 21 01:42 usr
drwxr-xr-x 2 <username> <group> 4.0K Mar 21 01:42 var
-rw-r--r-- 1 <username> <group>    2 Mar 21 01:42 VERSION
```
(眼熟)WDNMD,DNA动了,我不玩了:eggplant: :eggplant: :eggplant:.

/init文件是挂载内核后第一个调用的脚本,第一行
```#!/usr/bin/ash```
说明这个文件是用ash运行的,ash又是什么呢,看一下文件类型:
``` shell
$ file usr/bin/ash 
usr/bin/ash: symbolic link to busybox
```
Ash(Almquist shell)实际上是包含在busybox中的一个mininal shell.

Busybox是个挺神奇的应用,可以理解为一个捆绑了一堆应用的应用,调用内部命令的方式主要有两个:
``` shell
$ busybox [Function]
```
或者使用symbol/hard link链接到busybox,busybox会根据arg\[0\]的内容决定调用内部应用.

以上说的仅仅是Archlinux官方initramfs中的内容和/init文件的形式,如果是Centos PXE Boot版本的initrd.img,其中的内容则完全不同,/init文件也不再是一个busybox脚本:
``` shell
$ xz -dc < ./initrd.img | cpio -idmv
$ ls -lh
lrwxrwxrwx  1 <username> <group>    7 Mar 21 20:59 bin -> usr/bin
-rw-r--r--  1 <username> <group>  149 Oct 27 00:11 .buildstamp
drwxr-xr-x  2 <username> <group> 4.0K Oct 27 00:23 dev
drwxr-xr-x 14 <username> <group> 4.0K Mar 21 20:59 etc
lrwxrwxrwx  1 <username> <group>   23 Mar 21 20:59 init -> usr/lib/systemd/systemd
lrwxrwxrwx  1 <username> <group>    7 Mar 21 20:59 lib -> usr/lib
lrwxrwxrwx  1 <username> <group>    9 Mar 21 20:59 lib64 -> usr/lib64
drwxr-xr-x  2 <username> <group> 4.0K Oct 27 00:23 proc
drwxr-xr-x  2 <username> <group> 4.0K Oct 27 00:23 root
drwxr-xr-x  2 <username> <group> 4.0K Oct 27 00:23 run
lrwxrwxrwx  1 <username> <group>    8 Mar 21 20:59 sbin -> usr/sbin
-rwxr-xr-x  1 <username> <group> 3.1K Sep 30 23:57 shutdown
drwxr-xr-x  2 <username> <group> 4.0K Oct 27 00:23 sys
drwxr-xr-x  2 <username> <group> 4.0K Oct 27 00:23 sysroot
drwxr-xr-x  2 <username> <group> 4.0K Oct 27 00:23 tmp
drwxr-xr-x  9 <username> <group> 4.0K Mar 21 20:59 usr
drwxr-xr-x  3 <username> <group> 4.0K Mar 21 20:59 var
```
其中init文件指向的是systemd,甚至包含一个python2.7解释器.

### Init
init脚本是内核启动后执行的首个一个命令,作用也非常简单,就是启用一些必需的kernel module并将真正的root挂载到新的root目录下然后转移到真正的root下.
init最少要挂载root,甚至内核模块也不需要在这一步加载.

这里从gentoo wiki上抄了一个mininalistic init example:
``` shell
#!/bin/busybox sh
# Mount the /proc and /sys filesystems.
mount -t proc none /proc
mount -t sysfs none /sys

# Do your stuff here.
echo "This script just mounts and boots the rootfs, nothing else!"

# Mount the root filesystem.
mount -o ro /dev/sda1 /mnt/root

# Clean up.
umount /proc
umount /sys

# Boot the real thing.
exec switch_root /mnt/root /sbin/init
```

### Repack initramfs

打包initramfs非常简单,这样那样就好了:
``` shell
$ find . 2>/dev/null | cpio -o -c -R root:root > initramfs-new.img
$ zstd initramfs-new.img -o initramfs_new.img
```
initramfs\_new.img就是我们需要initramfs文件.

**相关资料:**
+ How to unpack/uncompress and repack/re-compress an initial ramdisk (initrd/initramfs) boot image file: [https://access.redhat.com/solutions/24029](https://access.redhat.com/solutions/24029)
+ Custom Initramfs: [https://wiki.gentoo.org/wiki/Custom_Initramfs#Extracting_the_cpio_archive](https://wiki.gentoo.org/wiki/Custom_Initramfs#Extracting_the_cpio_archive)