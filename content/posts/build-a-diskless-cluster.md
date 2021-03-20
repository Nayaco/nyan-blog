---
title: "Build a Diskless Cluster(Centos7)"
date: 2021-03-18T13:42:04+08:00
draft: false
tags: ["NFS", "Linux", "PXE", "dnsmasq"]
summary: "Build a Diskless Cluster, Using NFS, DNSMASQ"
---

无盘集群(Diskless Cluster),指集群中计算节点没有安装可启动(Bootable)的操作系统,无盘集群优点是维护方便,减少存储资源投入,使用MPI等方式执行计算时执行文件同步较方便;缺点是对
集群的网络性能,存储节点的IO性能要求较高.(PXE:M61:Media no found :upside_down_face:

本文参考USTC集群资料撰写.

目标集群拓扑结构:
```
|----------------|
|172.25.2.101(m0)|----
|----------------|    |
|172.25.2.102(s1)|----
|----------------|    |---|Ethernet Switch| 
|172.25.2.103(s2)|----
|----------------|    |
|172.25.2.104(s3)|----
|----------------|
```

### 准备工作

准备Centos7[安装镜像](https://mirrors.tuna.tsinghua.edu.cn/centos/7.9.2009/isos/x86_64/CentOS-7-x86_64-Minimal-2009.iso),此处地址为Tuna源.

如果使用IPMI/KVM安装镜像请自动跳过以下步骤:

准备USB flash installation medium.使用以下命令将镜像文件写入USB设备(使用USB设备/dev/sda,/dev/sdb...替换/dev/sdx):

``` shell
$ dd bs=4M if=/path/to/centos.iso of=/dev/sdx status=progress && sync
```
在m0节点上安装Centos,并升级系统,安装必要的包:

*dracut-network会为镜像内添加nfs等网络支持*

``` shell
$ yum -y update
$ yum -y install nfs-utils tftp-server dnsmasq syslinux dracut dracut-network
```
dracut-network包安装后在/etc/dracut.conf文件中添加:
``` shell
$ vim /etc/dracut.conf
# add nfs root support
add_dracutmodules+="nfs"
```
关闭SELinux
``` shell
$ vim /etc/sysconfig/selinux
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted 
$ setenforce 0
```
关闭防火墙:
``` shell
$ systemctl disable firewalld && systemctl stop firewalld
```
配置内部网卡:
``` shell
$ vim /etc/sysconfig/network-scripts/ifcfg-<LAN-NIC>
BOOTPROTO=static
IPADDR=172.25.2.101
NETMASK=255.255.255.0
ONBOOT=yes
$ ip addr add 172.25.2.101 dev <LAN-NIC>
```

### 配置TFTP和DHCP服务(dnsmasq)
创建tftp root目录(/srv/tftp)并修改/etc/dnsmasq.conf,添加TFTP相关配置,或者将配置文件放在/etc/dnsmasq.d下:
``` 
# Enable dnsmasq's built-in TFTP server
enable-tftp

# Set the root directory for files available via FTP.
tftp-root=/srv/tftp

# Do not abort if the tftp-root is unavailable
tftp-no-fail
```
配置PXE-BOOT相关配置项(Legacy):
```
# pxe boot linux
dhcp-boot=pxelinux/pxelinux.0
```
添加DHCP服务器相关配置:
```
# Only listen to routers' LAN NIC.  Doing so opens up tcp/udp port 53 to
# localhost and udp port 67 to world:
interface=<LAN-NIC>

# dnsmasq will open tcp/udp port 53 and udp port 67 to world to help with
# dynamic interfaces (assigning dynamic ips). Dnsmasq will discard world
# requests to them, but the paranoid might like to close them and let the
# kernel handle them:
bind-interfaces

# Dynamic range of IPs to make available to LAN pc
dhcp-range=172.25.2.50,172.25.2.100,12h

# If you’d like to have dnsmasq assign static IPs, bind the LAN computer's
# NIC MAC address:
dhcp-host=<MAC-S1>,s1,172.25.2.102
dhcp-host=<MAC-S2>,s2,172.25.2.103
dhcp-host=<MAC-S3>,s3,172.25.2.104
```
配置DNSMASQ开机自启动:
``` shell
$ systemctl enable dnsmasq && systemctl start dnsmasq
```

### 配置NFS相关服务
``` shell
$ mkdir /nfs /client_nodes
```
配置NFS共享:
``` shell
$ vim /etc/exports
/nfs            172.25.2.0/24(rw,async,no_root_squash)
/client_nodes   172.25.2.0/24(rw,async,no_root_squash)
/mnt            172.25.2.0/24(rw,async,no_root_squash,crossmnt)
$ systemctl restart nfs
$ exportfs -ra
```
配置NFS开机自启动:
``` shell
$ systemctl enable nfs
```
### 配置计算节点操作系统文件
安装可以使用rsync同步本地文件到计算节点或者安装新的系统,以下方法二选一:

+ 直接安装新的系统文件:
``` shell
$ yum install @Base kernel dracut-network nfs-utils --installroot=/nfs --releasever=/
```
这个方法安装的系统文件比较干净,孩子用了都说好.

+ rsync同步本地文件:
``` shell
$ rsync -a -e ssh --exclude='/proc/*' --exclude='/sys/*' / /nfs
```
这里需要修改网络配置:
``` shell
rm -f /nfs/etc/sysconfig/network-script/ifcfg-ens*
```
安装好系统文件后配置计算节点的挂载配置,修改/nfs/etc/fstab:
```
# none    /tmp        tmpfs   defaults    0 0 
tmpfs   /dev/shm    tmpfs   defaults    0 0 
sysfs   /sys        sysfs   defaults    0 0 
proc    /proc       proc    defaults    0 0
172.25.2.101:/nfs   /    nfs defaults,rsize=32768,wsize=32768,intr   1 1
172.25.2.101:/mnt   /mnt nfs defaults,rsize=32768,wsize=32768,intr   1 2
```
### 设置计算节点启动内核
复制本机vmlinuz文件到/srv/tftp目录,计算节点DHCP获取到IP后会从这里通过tftp下载:
``` shell
$ cp /boot/vmlinuz-<VMLINUZ-VERSION>.el7.x86_64 /srv/tftp
```
在/srv/tftp目录下生成支持NFS-root的initrd文件:
``` shell
$ dracut --add nfs /srv/tftp/initramfs-<VMLINUZ-VERSION>.el7.x86_64.img <VMLINUZ-VERSION>.el7.x86_64
$ chmod 644 /srv/tftp/initramfs-<VMLINUZ-VERSION>.el7.x86_64.img
```
上面的方法可能会出现诡异的问题,如果上面的方法不行,直接从源下载相应的[pxeboot文件](https://mirrors.tuna.tsinghua.edu.cn/centos/7.9.2009/os/x86_64/images/pxeboot/).

### 设置计算节点PXE引导文件
复制/usr/share/syslinux/pxelinux.0文件和vesamenu.c32到/srv/tftp目录:
``` shell
$ cp {/usr/share/syslinux/pxelinux.0,/usr/share/syslinux/vesamenu.c32} /srv/tftp/}
```
在TFTP根目录下建立pxelinux.cfg目录:
``` shell
$ mkdir -p /srv/tftp/pxelinux.cfg
```
设置计算节点boot默认配置文件/srv/tftp/pxelinux.cfg/default:
```
default vesamenu.c32
timeout 10
# Clear the screen when exiting the menu, instead of leaving the menu displayed.
# For vesamenu, this means the graphical background is still displayed without
# the menu itself for as long as the screen remains in graphics mode.
menu clear
menu title CentOS 7
menu vshift 8
menu rows 18
menu margin 8
#menu hidden
menu helpmsgrow 15
menu tabmsgrow 13

# Border Area
menu color border * #00000000 #00000000 none
# Selected item
menu color sel 0 #ffffffff #00000000 none
# Title bar
menu color title 0 #ff7ba3d0 #00000000 none
# Press [Tab] message
menu color tabmsg 0 #ff3a6496 #00000000 none
# Unselected menu item
menu color unsel 0 #84b8ffff #00000000 none
# Selected hotkey
menu color hotsel 0 #84b8ffff #00000000 none
# Unselected hotkey
menu color hotkey 0 #ffffffff #00000000 none
# Help text
menu color help 0 #ffffffff #00000000 none
# A scrollbar of some type? Not sure.
menu color scrollbar 0 #ffffffff #ff355594 none
# Timeout msg
menu color timeout 0 #ffffffff #00000000 none
menu color timeout_msg 0 #ffffffff #00000000 none
# Command prompt text
menu color cmdmark 0 #84b8ffff #00000000 none
menu color cmdline 0 #ffffffff #00000000 none

# Do not display the actual menu unless the user presses a key. All that is displayed is a timeout message.

menu tabmsg Press Tab for full configuration options on menu items.
menu separator # insert an empty line
menu separator # insert an empty line

label linux
  menu label ^Cloud CentOS 7
  menu default
  kernel vmlinuz-<VMLINUZ-VERSION>.el7.x86_64
  append root=/dev/nfs rw nfsroot=172.25.2.101:/nfs,rsize=32768,wsize=32768 ip=dhcp selinux=0\
    initrd=initramfs-<VMLINUZ-VERSION>.el7.x86_64.img biosdevname=0 net.ifnames=0 ipv6.disable=1

menu end
```
### 设置节点私有目录(optional)
除/var与/tmp目录外,基本都可以共享,考虑大IO应用(Gaussian),另外设定/tmp使用客户节点的本地硬盘.(USTC教的好)
``` shell
$ cp -a /nfs/var /client_nodes/s1
$ cp -a /nfs/var /client_nodes/s2
$ cp -a /nfs/var /client_nodes/s3
```
设置客户节点启动脚本:
``` shell
$ vim /nfs/etc/rc.local
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.
touch /var/lock/subsys/local
mount -o rw 172.25.2.101:/diskless/nodes/$HOSTNAME/var /var

mount /dev/sda /tmp
if [ $? != 0 ]; then
    mkfs.xfs /dev/sda
    # mkfs.ext4 /dev/sda
    mount /dev/sda /tmp
fi
```
设置计算节点的rc.local开机自启动:
``` shell
$ chroot /diskless/root
$ chmod +x /etc/rc.d/rc.local
$ systemctl enable rc-local
$ exit # ^D
```
### 后记
最后不要忘记修改计算节点root密码:
``` shell
$ chroot /nfs
$ passwd
```
其他的内容包括Infiniband,CUDA等安装在此处不多做介绍.

**相关资料:**
+ CentOS 7.6无盘集群设置:[http://hmli.ustc.edu.cn/doc/linux/centos7.6-diskless/](http://hmli.ustc.edu.cn/doc/linux/centos7.6-diskless/)
+ Configuring Dnsmasq to Support PXE Clients: [https://docs.oracle.com/cd/E37670_01/E41137/html/ol-dnsmasq-conf.html](https://docs.oracle.com/cd/E37670_01/E41137/html/ol-dnsmasq-conf.html)