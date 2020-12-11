---
title: "Creating minimal linux test environment"
date: 2020-12-06T20:29:49+01:00
draft: false
---

### Building Kernel with default x86_64 configuration

- create default kernel build configuration using existing *defconfig*
```
make ARCH=x86 x86_64_defconfig
```
The file *.config* is created in linux root directory. It can be still modified with
```
make menuconfig
```
- build kernel image
```
make -j8
```
As a result we obtain a *vmlinux* binary which is a stand-alone monolithic ELF image.
This binary contains no unresolved external refeces. Despite this file was found at the
top-level kernel directory. Very few platforms boot this image directly including qemu emulator.
In order to boot qemu we need to use bzImage compressed kernel file. From:
_arch/x86/boot/bzImage_

- preparing initramfs

I prepared a simple initramfs just to make kernel fully boot into shell. As a shell
it was used busybox (just remember to check if that's a standalone version "static")
The directories were created in compliance to FHS (Filesystem Hierarchy Standard).
And I finally I add an init script which initializes busybox and mount pseudo-file system like
sysfs procfs dev and tmp.

- running qemu
By running appropriate qemu command:
```
qemu-system-x86_64 -kernel ./arch/x86/boot/bzImage -nographic -append "console=ttyS0" -initrd initrd
```

In order to exit qemu run: _Ctrl-a x_

Full work can be found here: [test_environment](https://notabug.org/damaxi/test_environment)
