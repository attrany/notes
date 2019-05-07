---
title: 在PVE宿主机上访问KVM虚拟机镜像文件中的内容
date: 2017-03-02 14:54:27
tags: [KVM, 虚拟机, 镜像, images, LVM]
category: 工作日常
---

## 问题背景

在进行android项目构建尝试的过程中，遇到了一个棘手的问题。但是，在解决这个问题的过程中，引发了一个更为严重的结果。

在android构建过程中，aapt报错:
``` bash
libc.so.6: version `GLIBC_2.14` not found
```

运行命令，发现centos6下的libc版本只支持到2.12

``` bash
$ ldd --version
ldd (GNU) libc 2.12
```

然而，按照 此链接 所提供的信息升级libc库，安装完成并配置完环境变量*$LD_LIBRARY_PATH* 后，构建程序无响应，最后超时退出，看起来是卡在了什么地方。

操作过程中注意到这样一句话，但是当时没在意。
```
You cannot update glibc on Centos6 safely. 
```

后来经过查询，尝试按照 此链接 提供的回答，尝试更改软连接的指向。
``` bash
$ ln -s /opt/glibc-2.14/lib/libc.so.6 /lib64/libc.so.6
```

然后，``这台机器就挂了``。对，就是这么突然，怎么也没想到兼容问题会如此严重。

为了图方便，直接拿了测试环境的数据库主机来做实验，出现这种情况是始料未及的。出现问题后，因为服务不可用，好几个人过来找。捅了大篓子了 :(

所以，为了继(bao)续(zhu)革(fan)命(wan)，至少要把数据库找回来。

## 现象描述

- 软连接建立完成后，尝试运行构建，片刻后失去响应，然后所有终端断开连接。
- 在PVE管理平台上，看到该虚拟机处于io-error状态，CPU和内存图表数值陡将至极低，不波动。磁盘读写直接降为0。
- 尝试在PVE管理平台重启，重启报错，无法操作恢复虚拟机状态。
- 尝试登入宿主机，直接使用命令与宿主机通讯，依然报错。
- 尝试重启宿主机，重启完毕后，该虚拟机启动成功。但是，一直无法ping通，说明网络服务无法启动，无法远程连接。
- 全程慌得一比

## 解决过程

- 因为很笃定是由于修改libc的链接指向，导致问题的发生。所以解决问题的关键就是在于如何把这个文件再修改回去。
- 但是如何修改一个无法登入的KVM虚拟机下面的文件，以前从来没有接触过这方面的操作，只知道KVM虚拟机会有一个镜像文件，和VM和vbox做的虚拟机镜像文件类似。
- 登入宿主机，经过搜索发现了虚拟机镜像存储路径
    ``` bash
    /var/lib/vz/images/108/vm-108-disk-1.raw
    ```
    在这里补充一下，openvz虚拟机根目录的目录在如下路径，xxx为CT的ID
    ``` base
    /var/lib/vz/private/xxx/
    ```
- 找到了镜像文件，就考虑能否将该文件像虚拟光驱加载镜像一样挂载到某个目录，这样我们就可以进入这个镜像文件了，即使不能修改文件，至少可以把数据库文件拷贝出来
- 经过一番搜寻，发现可以先将镜像文件映射至循环设备（``loop device``），然后挂载至文件系统，具体过程如下：
    ``` bash
    # 查找可用的loop device
    root@pve:~# losetup -f
    /dev/loop0

    # 将镜像文件映射至该循环设备，也可以挂载img后缀的文件
    root@pve:~# losetup /dev/loop0 /var/lib/vz/images/108/vm-108-disk-1.raw

    # 因为急于求成，执行完本步骤后尝试挂载至文件系统，报错如下
    root@pve:~# mount /dev/loop0 /mnt/108
    mount: unknown filesystem type 'LVM2_member'

    # 读取虚拟设备的分区
    root@pve:~# kpartx -av /dev/loop0
    add map loop0p1 : 0 7727202 linear /dev/loop0 63
    add map loop0p2 : 0 449757 linear /dev/loop0 7727328

    # 查看卷（volume）状态
    root@pve:~# lvscan
    inactive          '/dev/vg_ta/lv_root' [28.37 GiB] inherit
    inactive          '/dev/vg_ta/lv_home' [25.26 GiB] inherit
    inactive          '/dev/vg_ta/lv_swap' [5.88 GiB] inherit
    
    # 此时挂载会报错
    root@pve:~# mount /dev/vg_ta/lv_root /mnt/108
    mount: special device /dev/vg_ta/lv_root does not exist
    
    # 激活所有的卷
    root@pve:~# vgchange -ay
    3 logical volume(s) in volume group "vg_ta" now active

    # 挂载成功
    root@pve:~# mount /dev/vg_ta/lv_root /mnt/108

    # 成功进入虚拟机文件系统
    root@pve:~# ll /mnt/108
    total 304
    dr-xr-xr-x.  26 root root   4096 Mar  2 10:33 .
    drwxr-xr-x    4 root root   4096 Mar  2 11:03 ..
    dr-xr-xr-x.   2 root root   4096 Mar  2 03:45 bin
    drwxr-xr-x.   2 root root   4096 Mar 20  2013 boot
    drwxr-xr-x.   2 root root   4096 Sep 24  2011 cgroup
    drwxr-xr-x.  16 root root   4096 Mar  1 18:17 data
    drwxr-xr-x.   2 root root   4096 Mar 20  2013 dev
    drwxr-xr-x. 103 root root  12288 Mar  2 10:33 etc
    drwxr-xr-x.   2 root root   4096 Mar 20  2013 home
    dr-xr-xr-x.  13 root root   4096 Mar  2 03:45 lib
    ...

    ```
- 到此，心已经放到肚子里了
    - 恢复libc软链接指向。
    - 重启虚拟机
    - 服务恢复正常

## 心得

- 实验性的操作要新开主机，不要嫌麻烦，否则真是得不偿失；
- 底层类库升级要慎重，如果需要新版本类库特性，最好使用新版操作系统；
- 数据库务必要备份，哪怕是开发或测试环境；

## 扩展知识

- linux系统中分区和卷的区别与联系
    - 分区是基于物理磁盘的，是系统文件管理时识别用的磁盘地址，分区表是直接写在 MBR 或者 GPT 里面的，相对于卷来说更为底层。
    - 卷是由LVM(Logical Volume Manager)进行管理，LVM是在磁盘分区和文件系统之间添加的一个逻辑层。通俗来讲就是把若干物理磁盘看为一个整体，在这个整体上进行逻辑分卷，以实现linux系统动态扩充磁盘容量的需求。
    
    ![LVM示意图](https://imgsa.baidu.com/baike/s%3D220/sign=0b4099b7b44543a9f11bfdce2e168a7b/8b13632762d0f70329d6d29f0bfa513d2697c509.jpg)

- 什么是循环设备(loop device)
    - 在类 UNIX 系统里，loop 设备是一种伪设备(pseudo-device)，或者也可以说是仿真设备。它能使我们像块设备一样访问一个文件。
    - 在使用之前，一个 loop 设备必须要和一个文件进行连接。这种结合方式给用户提供了一个替代块特殊文件的接口。因此，如果这个文件包含有一个完整的文件系统，那么这个文件就可以像一个磁盘设备一样被 mount 起来。
    - 上面说的文件格式，我们经常见到的是 CD 或 DVD 的 ISO 光盘镜像文件或者是软盘(硬盘)的 *.img 镜像文件。通过这种 loop mount (回环mount)的方式，这些镜像文件就可以被 mount 到当前文件系统的一个目录下。

## 参考资料

- [Accessing the contents of a KVM disk image](http://equivocation.org/node/107)
- [How to mount LVM volumes, help!](http://www.fedoraforum.org/forum/archive/index.php/t-64964.html)
- [mount: unknown filesystem type ‘LVM2_member’](http://pissedoffadmins.com/os/mount-unknown-filesystem-type-lvm2_member.html)
- [linux下使用kpartx挂载虚拟文件系统](http://www.cnblogs.com/Jer-/p/3330128.html)
- [loop设备及losetup命令介绍](http://blog.csdn.net/ustc_dylan/article/details/6878252)
- [LVM - 百度百科](http://baike.baidu.com/item/LVM)