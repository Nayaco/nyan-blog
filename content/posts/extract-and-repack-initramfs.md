---
title: "Extract and Repack Initramfs"
date: 2021-03-21T14:16:16+08:00
draft: false
tags: []
summary: ""
---
本文将拆解initrd/initramfs boot image,并介绍操作系统挂载内核后的启动过程.

initramfs是一种嵌入在操作系统内核中的根文件系统(root filesystem),在boot的早期阶段就会载入.

initramfs在完全启动后实际上并不会从内存中消失,[道理](https://www.linuxquestions.org/questions/linux-general-1/accessing-initrd-in-memory-after-boot-4175614932/?__cf_chl_jschl_tk__=3c085e1f32f31adec75525faf41b83178d0570bf-1616311609-0-AZj1LlCJkO4bsxNQm6gOccAtGIycIS-KJciOF3RCF59fzEWG7Su66dIdVpcN6QhWDBVODWFtm6aqPle4SIiL2JwLkR8uI-lvknKzG96qyNR75hCNjjugSDsnqL-qY4KpE5KIIT5cmsvL4sd3fr3D1DdL5gpux9MIhwPKKDBxfH_eS5ORM2PP_WfjZabU7fx_A1hjkeAvEhRJSrtF_vkTMDnd3a0QiGT1oPZUlcsVBx0bh1iE8jqgLjx61EojkBLuuZbnd_vXNBp5xHY_XmhBh02jZbQBEsvH2-1UvKg6r9R6BclHuql1lQuMoIS6NQLm4PPj45Bz0FBh1B6qRr-lP-JmMCmjBbOHLSamXaM8MjMATJC-ZlgpCr-iTc4YhRQ7_9eHFekwVue-j2RKoDjqbY8mHqKlbskrfBrsoZoxO-gsgoksBXK9BG0_McLpJykGyHsxD5izmOn1Hk_v5cmYZmeISBxrTAy5C7xvMt2RHDUr)上这个文件系统会位于/dev/initrd之类的设备下,但是并没有找到这个文件.

### 拆解initramfs

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
WDNMD,DNA动了,我不玩了:eggplant: :eggplant: :eggplant:.

/init文件是挂载内核后第一个调用的脚本,第一行
```#!/usr/bin/ash```
说明这个文件是用ash运行的,ash又是什么呢,看一下文件类型:
``` shell
$ file usr/bin/ash 
usr/bin/ash: symbolic link to busybox
```




