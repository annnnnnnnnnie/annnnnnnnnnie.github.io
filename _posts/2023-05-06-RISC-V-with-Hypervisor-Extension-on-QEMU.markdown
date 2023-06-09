---
layout: post
title: "Running RISC-V with Hypervisor Extension on QEMU"
categories: RISC-V Virtualization
---

# RISC-V, Virtualization and the Hypervisor Extension

This section will talk about RISC-V ISA, the role of virtualization, and the hardware support for virtualization.

# Running Linux Hypervisor and Linux Guest on QEMU

We will follow the following steps to cross compile the linux hypervisor and guest, compile a qemu with hypervisor extension and then run the hypervisor + guest on qemu.

1. [Environment used](#Environment)
2. [Getting the risc-v cross compilation toolchain](#getting-the-risc-v-cross-compilation-toolchain)
3. [Compile qemu-system-riscv64 that supports the hypervisor extension](#compile-qemu-system-riscv64-that-supports-the-hypervisor-extension)
4. [Use buildroot to build a linux hypervisor](#use-buildroot-to-build-a-linux-hypervisor)
5. [Build kvmtool](#build-kvmtool)
6. [Put everything together](#put-everything-together)
7. [Full boot](#full-boot)
8. [References](#references)


## Environment
Ubuntu 20.04 with sudo rights.
```
$ apt install gcc make g++ rsync unzip wget
```

## Getting the RISC-V Cross Compilation Toolchain
To cross compile for RISC-V linux, we need to use the cross compilation toolchain. It can be downloaded from [github](https://github.com/riscv-collab/riscv-gnu-toolchain). For our use case, it is suffice to download one from their releases, instead of compiling from source. The releases can be found [here](https://github.com/riscv-collab/riscv-gnu-toolchain/tags). We will look for one with name `riscv64-glibc-ubuntu-20.04-xxxxx-tar.gz`. You can download with `wget` and unzip with `tar -xvzf`.

> Note: If your OS is not Ubuntu20.04, you should look for a compatible release [here](https://github.com/riscv-collab/riscv-gnu-toolchain/tags).

```
$ wget https://github.com/riscv-collab/riscv-gnu-toolchain/releases/download/2023.04.29/riscv64-glibc-ubuntu-20.04-nightly-2023.04.29-nightly.tar.gz
$ tar -xvzf riscv64-glibc-ubuntu-20.04-nightly-2023.04.29-nightly.tar.gz
```

## Compile qemu-system-riscv64 that Supports the Hypervisor Extension
By default, qemu-system-riscv64 does not support the hypervisor extension. Hence we need to compile qemu from source.
First, get the essential packages following the [qemu wiki guide](https://wiki.qemu.org/Hosts/Linux). 

```
$ apt install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev ninja-build
```
It is possible to compile these packages from source and use them, but it would be quite complicated.

Get the repository:
```
$ git clone https://gitlab.com/qemu-project/qemu
$ cd qemu
$ git submodule init
$ git submodule update --recursive
```

Build qemu-system-riscv64 with hypervisor extension:
```
$ ./configure --target-list="riscv64-softmmu"
$ make -j $(nproc)
```
The built qemu executable will be under `./build/`.

We can check that h-extension is indeed enabled:
```
$ ./build/qemu-system-riscv64 -M virt -bios default -nographic
```
We can see that `Boot HART Base ISA        : rv64imafdch`, which includes `h`.

Use `ctrl-a` followed by `ctrl-x` to terminate qemu.

## Use buildroot to Build a Linux Hypervisor

There are many ways to build a linux kernel and a rootfs. Using buildroot is one of the less complicated option.

Get [Buildroot](https://buildroot.org/):
[Download build root](https://buildroot.org/download.html)

> Building linux kernel, busybox or buildroot use similar workflows. First `make xxxconfig` to generate a `.config` file. Then `make -j $(nproc)` to actually build it.

First, make sure that the `riscv-gnu-toolchain` is on our `PATH`.
```
$ PATH_TO_RISCV_TOOLCHAIN=/path/to/riscv-gnu-toolchain/bin
$ export PATH=$PATH_TO_RISCV_TOOLCHAIN:$PATH
```
Then use menuconfig to config.
```
$ export ARCH=riscv
$ export CROSS_COMPILE=riscv64-unknown-linux-gnu-
$ apt install libncurses-dev
$ make menuconfig
```
- Under `Target options`, set Target Architecture to riscv

- Under `Kernel`, press space to select the linux kernel. Use the latest kernel version. Under `Kernel configuration`, use the architecture default configuration.

- Under `Toolchain`, set toolchain type to external toolchain (Using the default option is fine, just that buildroot will download the toolchain for you). `Toolchain has xxx support` should also be configured accordingly. ![buildroot-toolchain-menuconfig](/images/RISCV-with-H-extension-QEMU/buildroot_menuconfig_toolchain.png)

- Under `System configuration`, set `Root filesystem overlay directories` to a folder on your host machine. The content of the folder will appear in the filesystem in the built image. We will call this `HOST_OVERLAY_DIR`

- Under `Filesystem images`, set the filesystem type to `ext4`. Increase the `exact size` for more spaces on the filesystem.

- Save and exit.

Now we can compile the kernel.
```
$ make -j $(nproc)
```

The kernel source as well as built items can be found under `./output/build/linux-?.?`. The kernel image will be under `arch/riscv/boot/Image` and the kvm module will be under `arch/riscv/kvm/kvm.ko`.

> Note: to rebuild the linux kernel and kernel modules, use (under buildroot folder):
```
$ make -j $(nproc) linux-rebuild
```

The autogenerated booting instruction can be found under `./ board/qemu/riscv64-virt/readme.txt`. However, this does not seem to work. Instead, we will use the following:
```
$ QEMU=/mnt/data/qemu/build/qemu-system-riscv64
$ BR_OUT=/mnt/data/buildroot-2023.02/output
$ $QEMU -nographic -machine virt \
    -m 2G \
    -kernel $BR_OUT/images/Image -append "root=/dev/vda ro console=ttyS0" \
    -drive file=$BR_OUT/images/rootfs.ext4,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0

```
The linux hypervisor should be able to boot with qemu. Use `root` as the login. Use `poweroff` to shutdown the linux hypervisor.

Next, we will run a guest linux on the hypervisor linux.

## Build kvmtool
Firstly build libfdt. This is to be installed under the cross compilation toolchain's sysroot, so apt installing won't work.
```
$ git clone git://git.kernel.org/pub/scm/utils/dtc/dtc.git
$ apt install bison flex
```

Install dtc to the riscv toolchain sysroot.
```
$ export ARCH=riscv
$ export CROSS_COMPILE=riscv64-unknown-linux-gnu-
$ export CC="${CROSS_COMPILE}gcc -mabi=lp64d -march=rv64gc"
$ TRIPLET=$($CC -dumpmachine)
$ SYSROOT=$($CC -print-sysroot)
$ make libfdt
$ make NO_PYTHON=1 NO_YAML=1 DESTDIR=$SYSROOT PREFIX=/usr LIBDIR=/usr/lib64/lp64d install-lib install-includes
```

> Note: check if $SYSROOT is pointing to the correct location. It should be something like `.../riscv/sysroot/`.

Then clone and build kvmtool
```
$ git clone https://git.kernel.org/pub/scm/linux/kernel/git/will/kvmtool.git
```

```
$ export ARCH=riscv
$ export CROSS_COMPILE=riscv64-unknown-linux-gnu-
$ make lkvm-static
```

> Note: You can edit the Makefile and point CC, LD OBJCOPY to the absolute location of the riscv toolchain cc, ld and objcopy.

## Put everything together

Recall that we have `HOST_OVERLAY_DIR` which allows us to put files onto the filesystem that the linux hypervisor is using. To boot a linux guest on top of it, we need some additional files.
- Image: This is the riscv64 linux kernel image that we will run as the guest OS.
- kvm.ko: This is the kernel module that we will insert into the hypervisor to enable kvm.
- lkvm-static: This is the vm manager that uses kvm to run a guest OS. 

Copy the abovementioned files to `HOST_OVERLAY_DIR` and rebuild using buildroot.

> Note: Use `find . -name linux -type d` in buildroot to locate the linux build dir. Copy over `arch/riscv/boot/Image` and `arch/riscv/kvm/kvm.ko`.

## Full boot

First boot the hypervisor. Make sure path to `Image` and `rootfs.ext4` is correct for your machine.
```
$ QEMU=/mnt/data/qemu/build/qemu-system-riscv64
$ BR_OUT=./br_output
$ $QEMU -nographic -machine virt \
    -m 2G \
    -kernel $BR_OUT/images/Image -append "root=/dev/vda ro console=ttyS0" \
    -drive file=$BR_OUT/images/rootfs.ext4,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0

```

Login with the user name `root`. `ls` to see if the files are here.
```
# ls
(expecting Image kvm.ko lkvm-static)
```
Insert the kvm module and boot the guest
```
# insmod kvm.ko
# ./lkvm-static run -m 256 -c1 --console serial -p "console=ttyS0 earlycon" -k ./Image --debug
```
Now the guest should be booting, with the host filesystem available at `/host`.

## References
This blog is mainly based on these existing sources. Links to documentations are also included.

1. [https://github.com/kvm-riscv/howto/wiki/KVM-RISCV64-on-QEMU](https://github.com/kvm-riscv/howto/wiki/KVM-RISCV64-on-QEMU)

There is also a repository that helps to organize steps mentioned in this post. It can be found [here](https://github.com/annnnnnnnnnie/linuxkvmriscv64). However, it is very fragile and full of global vars.