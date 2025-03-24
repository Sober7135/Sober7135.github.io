+++
date = '2025-03-19T03:12:45+08:00'
draft = true
title = 'Linux Kernel + QEMU + lldb + LLVM + VSCode = Debug Linux Kernel'
tags = ['Linux', 'Debug']
+++


# 内核 config

有两种方法,
1. 抄一下现在运行内核 config, 大概在 `/proc/config.gz`

```
zcat /proc/config.gz > .config
```
2. `make LLVM=1 defconfig`, 用 default config

1 的话编译时间会长很多

改一点编译选项, 运行 `make LLVM=1 menuconfig`
```
Processor type and features ->
  Randomize the address of the kernel image (KASLR) // N 关了, 出于安全要求, 会随机化binary的地址

Kernel hacking ->
  Compile-time checks and compiler options ->
    Compile the kernel with debug info ->                 // Y 选上
      DWARF version (Generate DWARF Version 5 debuginfo)  // Y 选上这个, 这样就会有debug information
      Reduce debugging information                        // N
      Provide GDB scripts for kernel debugging            // Y 这个也可以选上, 虽然我不知道这是干什么, 我应该也不会用 GDB 来调试, 但是不选摆布选
```

# 编译
视情况选择核心数, 不写就是没限制, 可能会报 OOM

```
make LLVM=1 -j
```

可能会报错, 原因是因为抄了 Linux 发行版的 config, 一般会把有 key 来验证啥的, 直接删了就好了. 类似与这种

```
CONFIG_MODULE_SIG_KEY="/var/tmp/portage/sys-kernel/gentoo-kernel-6.13.6/temp/kernel_key.pem"
```

生成 compile_commands 给 clangd, 然后在 vscode 就能随便跳, 但是肯定还是单步调试更好点
```
scripts/clang-tools/gen_compile_commands.py
```

# 调试

先试试, 

```bash
qemu-system-x86_64                \
-kernel arch/x86_64/boot/bzImage  \
-nographic                        \
-append "console=ttyS0"           
```

大概会有这样的报错
```
[    1.729731] md: ... autorun DONE.
[    1.730562] VFS: Cannot open root device "(null)" or unknown-block(0,0): error -6
[    1.730727] Please append a correct "root=" boot option; here are the available partitions:
[    1.731053] 0b00         1048575 sr0 
[    1.731116]  driver: sr
[    1.731316] Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
[    1.731562] CPU: 0 PID: 1 Comm: swapper/0 Not tainted 5.15.163 #5
[    1.731701] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS 1.16.3-20240910_120124-localhost 04/01/2014
[    1.731939] Call Trace:
[    1.732358]  <TASK>
[    1.732483]  dump_stack_lvl+0x5a/0x90
[    1.732754]  panic+0x111/0x2f0
[    1.732806]  mount_block_root+0x1f3/0x200
[    1.732925]  prepare_namespace+0xfb/0x170
[    1.732998]  kernel_init_freeable+0xfe/0x140
[    1.733078]  ? rest_init+0xb0/0xb0
[    1.733144]  kernel_init+0x11/0x110
[    1.733210]  ret_from_fork+0x22/0x30
[    1.733304]  </TASK>
[    1.733572] Kernel Offset: disabled
[    1.733755] ---[ end Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0) ]---
```
意思是没有 rootfs, 这里我们需要自己做一个 initramfs, 我这里采用的是 busybox

按照之前的 blog, 就可以创建了

```
[    1.164934] Run /init as init process
[    1.185535] busybox (68) used greatest stack depth: 14448 bytes left

Welcome to Micro Linux!
Boot took 1.17 seconds
```

但是我们发现现在的 Linux 是没有文件系统的, 这里我们需要创建一个, 然后 mount 上去

用 `qemu-img`/`dd` 创建一个 image, 然后格式化成 ext4
```
qemu-img create -f raw linux-disk.raw 10G    
mkfs.ext4 ./linux-disk.raw  
```

```bash
qemu-system-x86_64                                  \
-kernel arch/x86_64/boot/bzImage                    \
-initrd initramfs.cpio.gz                           \
-drive format=raw,file=linux-disk.raw               \
-append "console=ttyS0 root=/dev/sda nokaslr"       \
-m 2G                                               \
-s                                                  \
-S                                                  \
-net user,hostfwd=tcp::8888-:22                     \
-net nic,model=virtio                               \
-nographic                                          \
-qmp unix:./qmp-sock,server,nowait 2>&1 | tee log
```



# Reference
- [build with llvm](https://docs.kernel.org/kbuild/llvm.html)
- https://zhuanlan.zhihu.com/p/652682080
- https://www.kernel.org/doc/html/latest/process/debugging/gdb-kernel-debugging.html
- https://zhuanlan.zhihu.com/p/675453558
- https://nickdesaulniers.github.io/blog/2018/10/24/booting-a-custom-linux-kernel-in-qemu-and-debugging-it-with-gdb/
- https://cylab.be/blog/320/build-a-kernel-initramfs-and-busybox-to-create-your-own-micro-linux