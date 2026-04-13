+++
title = "Cross-compiling the Raspberry Pi Linux kernel with the PREEMPT_RT patch"
date = "2025-01-22"
draft = false

[taxonomies]
tags = ["raspberry pi", "linux", "kernel",]

[extra]
lang = "en"
+++

Compiling the Linux kernel on a Raspberry Pi is slow. This isn't surprising considering the Raspberry Pi is meant to run custom kernels, rather than build them. A better approach would be to cross-compile the kernel on a more powerful x86 machine and then transfer it to the Pi. This saves a lot of time, time that you can spend actually using the Pi for your projects, instead of waiting for the kernel to build.

My host machine in this case is a Thinkpad T480 with a i7-8650U running Ubuntu 22.04.5 LTS. The kernel takes around 30 minutes to compile on this machine (as opposed to ~forever on the RPi). My target is the Raspberry Pi CM4 also running also Ubuntu 22.04 LTS.

## Setting up the build environment

Firstly install the build dependencies.

```bash
$ sudo apt install build-essential libgmp-dev libmpfr-dev libmpc-dev libisl-dev libncurses5-dev bc git-core bison flex texinfo libssl-dev make gcc-aarch64-linux-gnu binutils
```

This installs the required build dependencies as well as the cross compiler. Now download the kernel from Raspberry Pi's Github.

```bash
$ git clone https://github.com/raspberrypi/linux.git
```
This is a pretty large download ~4GB.

In order to patch the kernel we need to ensure that our kernel versions match. For example, if raspberrypi/linux has a merge commit for linux 6.6.35, there should be an existing rt patch set for 6.6.35.

```bash
$ cd linux
$ git checkout -b rpi-6.6-rt --track origin/rpi-6.6.y
```

Find the exact kernel version from the Makefile. In this case the current kernel version is 6.6.78.

```bash
$ cat Makefile 
# SPDX-License-Identifier: GPL-2.0
VERSION = 6
PATCHLEVEL = 6
SUBLEVEL = 78
EXTRAVERSION =
NAME = Pinguïn Aangedreven
...
```

Find the corresponding rt patch set from [here](https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/). Download and apply the patch.

```bash
$ wget https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/6.6/patch-6.6.78-rt52-rc1.patch.xz
$ cd linux
$ xzcat ../patch-6.6.35-rt34.patch.xz | patch -p1
```

## Configuring and Compiling the kernel

Before the configuring the kernel, we need to export a few environment variables, so that the compilation is successful.
```bash
$ export ARCH=arm64
$ export KERNEL=kernel8
$ export CROSS_COMPILE=aarch64-linux-gnu-
```
Generate the default configuration using
```bash
$ make bcm2711_defconfig
```
Then
```bash
$ make menuconfig
```
This opens up a ncurses menu from where we can configure our kernel before compilation.
- Go to *General -> Preemption Model* and select *Fully Preemptible Kernel (Real-Time)*
![preemption model](/img/preemption-model.png)
- I set *Kernel Features -> Timer frequency* to 1000 HZ
![timer freq](/img/timer-freq.png)
- *CPU Power Management -> CPU Frequency Scaling -> Default CPUFreq governor* to "performance"
![cpufreq](/img/cpufreq-governor.png)
- I disabled *Virtualization*
Finally save this as .config and exit. We can now go ahead and compile the kernel.

```bash
$ make -j$(nproc) Image modules dtbs
```

## Installation to Raspberry Pi target

I installed the custom kernel on Raspberry Pi Compute Module 4. I have access to an IO board, so I installed using [rpiboot](https://github.com/raspberrypi/usbboot).

After running rpiboot, the system-boot and writable partitions will be mounted on /dev/sda and /dev/sdb. If you are using a SD card to load the kernel on to a Raspberry Pi, you will be moving it to the same partitions.

Copy the kernel and DTBs onto the drive:

```bash
$ sudo cp /media/$USER/system-boot/vmlinuz /media/$USER/system-boot/vmlinuz.old #backup the old kernel
$ sudo cp arch/arm64/boot/Image /media/$USER/system-boot/vmlinuz
$ sudo cp arch/arm64/boot/dts/broadcom/*.dtb /media/$USER/system-boot/
$ sudo cp arch/arm64/boot/dts/overlays/*.dtb* /media/$USER/system-boot/overlays/
$ sudo cp arch/arm64/boot/dts/overlays/README /media/$USER/system-boot/overlays/
```

Install the kernel modules onto the drive:

```bash
$ sudo make modules_install INSTALL_MOD_PATH=/media/$USER/writable
```

Check for the kernel using `uname`.
```bash
$ uname -a
Linux a3cu9 6.6.35-rt34-rpi-v8+ #1 SMP PREEMPT_RT Tue Aug  6 04:29:19 PDT 2024 aarch64 aarch64 aarch64 GNU/Linux
```

## Conclusion

Hence, cross-compiling the Raspberry Pi Linux kernel with the PREEMPT_RT patch opens up a realm of possibilities for developers aiming to enhance real-time performance on their Raspberry Pi systems.

{% note(header="Note") %}
Linux 6.12 has merged the PREEMPT_RT patch. This is enabled for ARM64, RISCV as well as x86. This enables the ability to configure and compile the kernel natively without having to patch it. Interestingly enough, the main blocker was the printk function used to log messages from the kernel to the console or system logs.
{% end %}