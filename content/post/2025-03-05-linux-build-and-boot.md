---
layout:     post 
title:      "Build and boot Linux"
subtitle:   "Simple build for aarch64 and booting with QEMU"
date:       2025-03-05
author:     "Gabor Toth"
URL:        "/2025/03/05/linux-build-and-boot/"
image:      "img/post-bg-coffee.jpeg"
draft:      false
---

## Why Linux?
Nowadays, in the market of operating systems – including embedded devices – Android is leading by a wide margin. In January 2025, it reached a market share of 46.18%, while the second-place contender, Windows, held just over half of that, at 25.46%.
Since the core of the Android operating system is the Linux kernel, it can be stated that Linux-based systems run on nearly half of all the devices.

<div style="display: flex; justify-content: center; align-items: center;">
    <div id="all-os_combined-ww-monthly-202401-202501" style="width:600px; height:400px;"></div>
</div>
<p style="text-align: center;">Source: <a href="https://gs.statcounter.com/os-market-share">StatCounter Global Stats - OS Market Share</a></p>
<script type="text/javascript" src="https://www.statcounter.com/js/fusioncharts.js"></script>
<script type="text/javascript" src="https://gs.statcounter.com/chart.php?all-os_combined-ww-monthly-202401-202501&chartWidth=600"></script>

The popularity of Linux can be attributed to several factors:

- Open-source and free: It is freely downloadable, modifiable, and distributable, allowing users to tailor it to their specific needs.
- Stability and reliability: The system ensures seamless operation with minimal downtime.
- Community support: The Linux community is responsible for its development and maintenance, providing opportunities for anyone to contribute, while not being solely driven by corporate interests.
- Security: Bugs are fixed quickly, and due to the significantly lower incidence of viruses compared to, for example, Windows, it offers greater protection.
- Flexible customization: Any component can be replaced (e.g., GNOME, KDE), and there are many distributions to choose from, enabling full personalization.
- Resource management: Linux handles resources in a highly efficient and fast manner, ensuring high performance.

Since Linux plays a key role in personal computers and embedded systems, such as IoT devices, smartphones, network devices, and other low-power, server-type systems, its knowledge has become essential for those who wish to work with embedded systems.

<img src="/img/post/2025-03-05-linux-build-and-boot/tux.png" alt="Oh no the image is missing..." style="max-width: 200px; height: auto;" />

## Kernel vs distribution
The Kernel (or Linux Kernel) is the heart and foundation of the Linux operating system. It is a low-level software that communicates directly with hardware components such as the processor, memory, hard drives, and other devices. The kernel is responsible for managing system resources like memory and CPU, scheduling different programs, and ensuring smooth communication between the operating system and hardware elements. The kernel does not provide user applications or graphical interfaces by itself; it only ensures the basic, low-level functionality.

A Linux distribution (or simply distro) is a complete operating system that, in addition to the Linux kernel, includes other software to provide full functionality for users. A distribution contains the kernel along with essential applications and tools, such as a graphical interface, package manager, and other stuff.

## Aim of this post
My goal is to provide a hands-on experience in downloading, configuring, compiling, and booting a minimalist Linux kernel, enabling you to explore the topics covered in the upcoming posts in a practical way.

## Environment setup
The machine I used is a Macbook with *Apple M3 Pro* CPU, which has ARM 64-bit architecture (aarch64). If you are on a different architecture (e.g x86), you should install a different version of QEMU and use different Linux config.
You can check the architecture of your processor with the *uname -m* command.

For this project I set up a virtual machine with <a href="https://ubuntu.com/download/desktop">Ubuntu</a> 24.04 LTS.

After installation update the system
```bash
sudo apt-get update
sudo apt-get upgrade
```

Get toolchain for compilation (like <a href="https://gcc.gnu.org/">GCC</a>, <a href="https://www.gnu.org/software/make/">GNU Make</a>):
```bash
sudo apt-get install build-essential
```

## Prepare the kernel
Install the dependencies
```bash
sudo apt-get install flex bison libncurses-dev libssl-dev
```

Get the source
```bash
git clone --depth=1 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git  
cd linux
```

(Optional) If you wish to work on the exact same version as I did:
```bash
git checkout 27102b38b8ca7ffb1622f27bcb41475d121fb67f
```

Generate a default configuration depending on your architecture
```bash
make ARCH=arm64 defconfig
```

Compile the kernel
```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

If there were no errors, a bootable image file is created at:
`linux/arch/arm64/boot/Image`

## Create filesystem with <a href="https://buildroot.org/">Buildroot</a>
A filesystem is essential to boot Linux (or any modern operating system) because it provides the structure and organization needed to store and access all the necessary files, programs, and data required during the boot process. Let's create one with Buildroot (will take some time):

Get the source
```bash
git clone https://git.buildroot.net/buildroot
cd buildroot
make menuconfig
```
<img src="/img/post/2025-03-05-linux-build-and-boot/menuconfig.png" alt="Oh no the image is missing..." style="max-width: 600px; height: auto;" />

Set your target architecture (*lscpu* command helps). In my case:
`Target options / Target Architecture / ARM (little endian)`

Enable the creation of CPIO root filesystem
`Filesystem images / cpio the root filesystem (for use as an initial RAM filesystem)`

Being able to debug is always a good start...install the <a href="https://www.sourceware.org/gdb/">GNU Debugger</a> into the kernel image
```bash
Toolchain / Enable C++ Support
Target Packages / Debugging, profiling and benchmark / gdb
```

Save and exit

Build Buildroot
```bash
make -j$(nproc)
```

The filesystem image file will be accessible in
`buildroot/output/images/rootfs.cpio`


## Let <a href="https://www.qemu.org/">QEMU</a> roar!
QEMU is an emulation tool that allows you to run the image. 
```bash
sudo apt-get install qemu-system-aarch64
```

Find the CPU you are running on (google or `qemu-system-aarch64 -cpu help`), and run
```bash
qemu-system-aarch64 \
    -M virt -cpu cortex-a57 -smp 4 -m 1024 \
    -kernel linux/arch/arm64/boot/Image \
    -initrd buildroot/output/images/rootfs.cpio \
    -nographic -append "console=ttyAMA0"
```

Where:
- `-M virt`: A QEMU-provided generic virtual board that offers good compatibility with various peripherals.
- `-cpu cortex-a57`: Specifies the CPU model, defining the instruction set and available features.
- `-smp 4`: Enables Symmetric Multi-Processing (SMP), allowing the guest system to use four virtual processors.
- `-m 1024`: Allocates 1 GB of RAM to the virtual machine.
- `-kernel arch/arm64/boot/Image`: Specifies the kernel image file to boot from.
- `-initrd rootfs.cpio`: Loads the root filesystem created by Buildroot. Change the path or copy the file to the Linux root directory if needed.
- `-nographic`: Disables graphical output and redirects input/output to the serial console.
- `-append "console=ttyAMA0"`: Passes the console= boot argument to the kernel, setting the primary console to ttyAMA0, a virtual serial port commonly used in ARM architectures.

After a quick boot you can log in with `root` password and enjoy your brand new Linux kernel.

<img src="/img/post/2025-03-05-linux-build-and-boot/qemu-linux-boot.png" alt="Oh no the image is missing..." style="max-width: 600px; height: auto;" />
