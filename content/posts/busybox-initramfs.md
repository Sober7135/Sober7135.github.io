+++
date = '2025-03-19T17:23:58+08:00'
draft = true
title = '[踩坑] Gentoo + busybox + initramfs = 💥'
+++

有点标题党, 应该是 `-march=native` + busybox + initramfs = 💥

# background 

最近打算 debug Linux 玩玩, 编译很简单, 但是在制作 initramfs 可一点也不简单, 记录一下踩坑的记录

首先是一些基本信息
```
OS: Gentoo Linux
make.conf:
...
COMMON_FLAGS="-march=native -O3 -pipe -flto"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
...
```

秉承了快就是好的原则, 我直接开了 `-march=native`, 整个系统都是用这个来构建的, 除了 kernel, 因为我懒得搞 menuconfig

# Gentoo 上编译 busybox

制作 initramfs 一个主流的方案是 busybox, 在网上也是很轻松找了资料[^1]

简单把 busybox 的 static USE Flag 加上, 编译一下

```bash
mkdir -p initramfs/bin initramfs/sbin initramfs/etc initramfs/proc initramfs/sys initramfs/dev initramfs/usr/bin initramfs/usr/sbin
cp -a /bin/busybox initramfs/bin/
vi initramfs/init
```
init 内容为
```bash
#!/bin/busybox sh

/bin/busybox mount -t devtmpfs devtmpfs /dev # 这里不知道为啥要加上 /bin/busybox
/bin/busybox mount -t proc none /proc
/bin/busybox mount -t sysfs none /sys

cat <<!

Welcome to Micro Linux!
Boot took $(cut -d' ' -f1 /proc/uptime) seconds

!
exec sh
```
```bash
chmod +x initramfs/init
cd initramfs
# 生成 initramfs
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.cpio.gz
cd ..
```

qemu 启动! 
```
[    1.182234] Run /init as init process
[    1.191694] traps: init[1] trap invalid opcode ip:35c575 sp:7ffd16577fc0 error:0 in busybox[276000+273000]
[    1.193842] Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000004
```

这错误好像之前在尝试用 valgrind 检查我写的 C++ 程序时候遇到过, 原因就是因为`-march=native`. 我简单思考了一下, 似乎我用 `-mtune=generic` 编译就行了, 对对对..对吗??

对的对的. 接下来就需要修改编译 busybox 的选项
```
mkdir -p /etc/portage/env/sys-apps/
# 创建一个单独的 env 用来编译 busybox
sudoedit /etc/portage/env/sys-apps/busybox 
```

加上这个
```
COMMON_FLAGS="-mtune=generic -O3 -flto"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
```

```bash
mkdir -p /etc/portage/package.env/sys-apps/
# 为 sys-apps/busybox 指定 env
sudoedit /etc/portage/package.env/sys-apps/busybox  
```

加上这个
```
sys-apps/busybox sys-apps/busybox
```

重新编译, 编译完用 `emerge --info busybox` 看一下编译的选项
```bash
sys-apps/busybox-1.36.1-r3::gentoo was built with the following:
USE="static -debug -livecd -make-symlinks -math -mdev -pam -savedconfig (-selinux) -sep-usr -syslog -systemd" ABI_X86="(64)"
CFLAGS="-mtune=generic -O3 -flto"
CXXFLAGS="-mtune=generic -O3 -flto"
```

一顿操作之后, 还是一样的错误 😭, 发现因为整个系统都是 `-march=native` 的, 链接的库也还是 `-march=native` 的, 这下完了, 只能重新用 docker 编译了

## 在 Arch Linux docker 上编译/偷 busybox

首先是 download source code, `tar`, 没啥好说, 接下来需要 `make menuconfig` 改成 static link
遇到了这个问题

```
 *** Unable to find the ncurses libraries or the
 *** required header files.
 *** 'make menuconfig' requires the ncurses libraries.
 *** 
 *** Install ncurses (ncurses-devel) and try again.
 *** 
make[2]: *** [/root/busybox/busybox-1.37.0/scripts/kconfig/lxdialog/Makefile:15: scripts/kconfig/lxdialog/dochecklxdialog] Error 1
make[1]: *** [/root/busybox/busybox-1.37.0/scripts/kconfig/Makefile:14: menuconfig] Error 2
```
解决方法[^2], why 
>This issue is caused by the fact that GCC14 introduces breaking changes and forces -Werror=implicit-int by default. 

接着开一下 static, `make`, 又炸了
```
...
networking/tc.c:319:57: error: invalid use of undefined type ‘struct tc_cbq_wrropt’
  319 |                                 printf("allot %ub ", wrr->allot);
...
```
呃呃呃, 还好偶然发现 Arch Linux 的 busybox 是 static 的, 下载之后把它偷过来就好了 

```
[    1.164934] Run /init as init process
[    1.185535] busybox (68) used greatest stack depth: 14448 bytes left

Welcome to Micro Linux!
Boot took 1.17 seconds
```



# Reference
[^1]: https://cylab.be/blog/320/build-a-kernel-initramfs-and-busybox-to-create-your-own-micro-linux
[^2]: https://stackoverflow.com/questions/78491346/busybox-build-fails-with-ncurses-header-not-found-in-archlinux-spoiler-i-alrea 