本篇对RISC-V操作系统的启动描述较为基础，结合了D1芯片的部分特色，适合了解原理后制作烧录镜像。
<!-- more -->

# RISC-V模式
RISC-V有三种模式，随系统的启动进行切换：
- M-mode(Machine Mode) ：ZSBL、FSBL、BBL
- S-mode(Supervisor Mode)：OS、U-Boot
- U-mode(User Mode)：User

# 启动流程
D1芯片从上电开始从0x0000 0000启动一个BROM(Boot ROM)，这是固化在芯片ROM中的一段引导程序，开始进入bootloader下阶段，否则进入fel模式。BROM是Boot Loader的最初阶段，Zeroth Stage Boot Loader(ZSBL)。
## Boot0(FSBL)
从这里开始就是我们烧录在tf卡（闪存）上的内容了。D1芯片上BROM读取boot0的位置在0x0002 0000，即从128K的位置上开始。
### SPL引导
U-Boot 分为 uboot-spl 和 uboot 两个组成部分。SPL 是 Secondary Program Loader 的简称，第二阶段程序加载器。对于嵌入式SOC(System On Chip)芯片来说，芯片内本身含有SRAM用于将闪存中的bootloader(uboot)加载到RAM来运行，但由于片内RAM大小限制，加载不了完整的U-boot程序，所以需要在片外的RAM上运行完整的Uboot。
作为bootloader的第一阶段FSBL(First Stage Boot Loader)，BOOT0从启动日志上可以看出一些它的功能：打开倍频统一时钟，初始化串口，DRAM内存初始化测试，储存（闪存）初始化测试，标记三个程序(文件)入口：opensbi、DTB（设备树）、u-boot，将内存信息加载到设备树，跳转至bootloder下一阶段。在片外DRAM上加载OpenSBI与Uboot本体，并跳转至OpenSBI。
```
[66]HELLO! BOOT0 is starting!
[69]BOOT0 commit : 882671f-dirty
[72]set pll start
[74]periph0 has been enabled
[77]set pll end
[78]board init ok
[80]DRAM only have internal ZQ!!
[83]get_pmu_exist() = -1
[85]ddr_efuse_type: 0x0
[88][AUTO DEBUG] two rank and full DQ!
[92]ddr_efuse_type: 0x0
[95][AUTO DEBUG] rank 0 row = 15
[98][AUTO DEBUG] rank 0 bank = 8
[101][AUTO DEBUG] rank 0 page size = 2 KB
[105][AUTO DEBUG] rank 1 row = 15
[108][AUTO DEBUG] rank 1 bank = 8
[111][AUTO DEBUG] rank 1 page size = 2 KB
[115]rank1 config same as rank0
[118]DRAM BOOT DRIVE INFO: V0.24
[121]DRAM CLK = 792 MHz
[123]DRAM Type = 3 (2:DDR2,3:DDR3)
[126]DRAMC ZQ value: 0x7b7bfb
[129]DRAM ODT value: 0x42.
[131]ddr_efuse_type: 0x0
[134]DRAM SIZE =1024 M
[138]DRAM simple test OK.
[140]dram size =1024
[142]card no is 0
[144]sdcard 0 line count 4
[146][mmc]: mmc driver ver 2021-04-2 16:45
[156][mmc]: Wrong media type 0x0
[159][mmc]: ***Try SD card 0***
[167][mmc]: HSSDR52/SDR25 4 bit
[170][mmc]: 50000000 Hz
[172][mmc]: 59356 MB
[174][mmc]: ***SD/MMC 0 init OK!!!***
[221]Loading boot-pkg Succeed(index=1).
[225]Entry_name        = opensbi
[228]Entry_name        = dtb
[230]Entry_name        = u-boot
[234]Adding DRAM info to DTB.
[239]Jump to second Boot.
```
### 其他引导
除了Uboot-SPL引导还可以通过Coreboot引导FSBL阶段，使其经过完整的bootloader阶段进入内核。

## OpenSBI
### SBI
RISC-V SBI是一种标准，是riscv架构独有的概念。OpenSBI 是 RISC-V SBI 规范的一种 C 语言实现。SBI作为Bootloader中的一个阶段，BBL(Berkeley Boot Loader)，提供加载，并且管理着二进制接口，实际上提供了S-mode模式对M-mode模式的调用，作为系统管理硬件的抽象接口。OpenSBI在引导后并不结束，而是作为系统于硬件交互的桥梁一直运行于后台。

### OpenSBI启动
opensbi提供了三种引导启动模式
- FW_PAYLOAD：Opensbi固件与uboot等绑定在一起
- FW_JUMP：直接跳转到bootloader下阶段
- FW_DYNAMIC：跳转的时候会传递动态的参数

全志芯片提供工具将opensbi、DTB、u-boot打包成一个toc1阶段，故在OpenSBI启动后会跳转至Uboot本体。
```
OpenSBI v1.0-104-ge6793dc
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name             : Sipeed Lichee RV Dock
Platform Features         : medeleg
Platform HART Count       : 1
Platform IPI Device       : ---
Platform Timer Device     : --- @ 0Hz
Platform Console Device   : uart8250
Platform HSM Device       : sun20i-d1-ppu
Platform Reboot Device    : sunxi-wdt-reset
Platform Shutdown Device  : ---
Firmware Base             : 0x40000000
Firmware Size             : 240 KB
Runtime SBI Version       : 1.0

Domain0 Name              : root
Domain0 Boot HART         : 0
Domain0 HARTs             : 0*
Domain0 Region00          : 0x0000000040000000-0x000000004003ffff ()
Domain0 Region01          : 0x0000000000000000-0xffffffffffffffff (R,W,X)
Domain0 Next Address      : 0x000000004a000000
Domain0 Next Arg1         : 0x0000000044000000
Domain0 Next Mode         : S-mode
Domain0 SysReset          : yes

Boot HART ID              : 0
Boot HART Domain          : root
Boot HART Priv Version    : v1.11
Boot HART Base ISA        : rv64imafdcvx
Boot HART ISA Extensions  : time
Boot HART PMP Count       : 8
Boot HART PMP Granularity : 2048
Boot HART PMP Address Bits: 38
Boot HART MHPM Count      : 0
Boot HART MIDELEG         : 0x0000000000000222
Boot HART MEDELEG         : 0x000000000000b109
```

## U-Boot
UBoot是一种支持Risc-V架构的流行bootloader。作为引导程序的最后一阶段，来引导内核加载进入DRMA运行。实际上Uboot也可以视为是一个小型系统，含有各种驱动程序、例如Uart串口驱动、网口驱动、屏幕驱动等，可以运行有限的命令，例如分区操作、执行指定地址程序、tftp传输等。也因此从M-mode切换到S-mode。在Uboot载入后，等待3s（默认设置）未进行任何操作，则执行默认操作而不显示等待输入命令的shell（类似于BIOS）。这个默认的操作就是将内核程序放入内存并执行，通过以下log可以观察到这一具体步骤：扫描当前闪存设备上的extlinux.conf内核设置文件与Image内核二进制文件、将设备树文件交给内核管理、运行内核。Uboot并不于Linux内核同时运行，它为内核单向提供所需要的参数。
```
U-Boot 2022.07-rc3-35470-gafc07cec42-dirty (Oct 25 2022 - 14:18:35 +0000)

CPU:   rv64imafdc
Model: Sipeed Lichee RV Dock
DRAM:  1 GiB
sunxi_set_gate: (CLK#24) unhandled
Core:  51 devices, 21 uclasses, devicetree: board
WDT:   Started watchdog@6011000 with servicing (16s timeout)
MMC:   mmc@4020000: 0, mmc@4021000: 1
Loading Environment from nowhere... OK
In:    serial@2500000
Out:   serial@2500000
Err:   serial@2500000
Net:   No ethernet found.
Hit any key to stop autoboot:  0

switch to partitions #0, OK
mmc0 is current device
Scanning mmc 0:1...
Found /extlinux/extlinux.conf
Retrieving file: /extlinux/extlinux.conf
1:      default
Retrieving file: /extlinux/../Image
append: earlycon=sbi console=ttyS0,115200n8 root=/dev/mmcblk0p2 rootwait cma=96M
Moving Image from 0x40040000 to 0x40200000, end=415ec1e8
## Flattened Device Tree blob at 7fb13e90
   Booting using the fdt blob at 0x7fb13e90
   Loading Device Tree to 0000000042df6000, end 0000000042dfff3e ... OK

Starting kernel ...
```

## Kernel
从这里开始才是进入Linux操作系统的第一步，在自检后启动了Linux第一个进程`/sbin/init`，所有进程都由它的子进程分化而来。进程启动后开始加载操作系统的各种功能：进程管理、储存管理、设备管理、文件系统等。这些系统级的程序依然运行于S-mode模式下，作为Linux操作系统的最底层运行。
```
Starting kernel ...

[    0.000000] Linux version 5.19.0-rc1-gfe178cf0153d-dirty (root@cirrus-task-6386852725784576) (riscv64-linux-gnu-gcc (Ubuntu 12.1.0-2ubuntu1~22.04) 12.1.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #1 PREEMPT Tue Oct 25 14:14:36 UTC 2022
[    0.000000] OF: fdt: Ignoring memory range 0x40000000 - 0x40200000
[    0.000000] Machine model: Sipeed Lichee RV Dock
...
[    3.241454] Run /sbin/init as init process
[    4.167677] systemd[1]: System time before build time, advancing clock.
[    4.274446] systemd[1]: systemd 250.4-1 running in system mode (+PAM +AUDIT +SELINUX +APPARMOR +IMA +SMACK +SECCOMP +GCRYPT +GNUTLS -OPENSSL +ACL +BLKID +CURL +ELFUTILS +FIDO2 +IDN2 -IDN +IPTC +KMOD +LIBCRYPTSETUP +LIBFDISK +PCRE2 -PWQUALITY -P11KIT -QRENCODE +BZIP2 +LZ4 +XZ +ZLIB +ZSTD -BPF_FRAMEWORK -XKBCOMMON +UTMP +SYSVINIT default-hierarchy=unified)
[    4.306347] systemd[1]: Detected architecture riscv64.

Welcome to Deepin 23!

[    4.328693] systemd[1]: Hostname set to <deepin-riscv>.
[    6.404994] systemd[1]: Queued start job for default target Graphical Interface.
[  OK  ] Created slice Slice[    6.423109] systemd[1]: Created slice Slice /system/getty.
 /system/getty.
[  OK  ] Created slice Slice[    6.450847] systemd[1]: Created slice Slice /system/modprobe.
 /system/modprobe.
 ...
[  OK  ] Reached target Graphical Interface.
         Starting Record Runlevel Change in UTMP...
[  OK  ] Finished Record Runlevel Change in UTMP.
[  OK  ] Started Hostname Service.

[   31.981895] fbcon: Taking over console

Deepin GNU/Linux
Deepin GNU/Linux 23 deepin-riscv hvc0

deepin-riscv login:  23 deepin-riscv ttyS0

[   36.559043] IPv6: ADDRCONF(NETDEV_CHANGE): wlan0: link becomes ready

deepin-riscv login:
```

## Rootfs
Linux系统全称为GNU/Linux，GNU指自由软件项目，Linux内核也可以简单视为一个程序软件。一个完整的操作系统，除了最核心的系统级的程序以外，管理与使用计算机资源的其他用户级的软件同样重要。对这些用户级程序进行管理、打包的工具与软件源加上Linux内核就组成了一个完整的Linux发行版，例如Debian、Ubuntu、CentOS、Deepin等。这些发行版在系统管理方面没什么本质区别，只有在桌面环境、安装软件时才会有一些差异。
这些预装的软件最终都会存为一个根文件系统(rootfs)，即进入操作系统后的最底层目录`/`。如果没有找到这个文件系统中的init程序，内核的最终运行会报错`Kernel panic`：
```
[    1.386099] Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance.
```

挂载根文件系统需要通过虚拟文件系统，简称 VFS（Virtual Filesystem），一个内核软件层。有两种常用的方式挂载
- 闪存挂载：通过本地emmc闪存中的根文件系统挂载。
- 网络挂载：利用NFS（Network File System）协议，通过网络上的根文件系统挂载。

挂载完成预装软件与系统配置的rootfs后，就可以对系统进行正常的操作与使用了。

# 总结
流程总体来说是BROM->boot0(spl)->opensbi->U-Boot->Kernel-Rootfs。其中全志芯片将boot0的spl作为toc0阶段，opensbi、dtb、uboot合并作为toc1阶段。
![启动流程](https://file.elecfans.com/web1/M00/D7/D3/pIYBAF_pQsiARvy5AAGTzX9cMiI738.png)
![启动流程2](https://file.elecfans.com/web1/M00/D7/D3/pIYBAF_pQsiAMQnBAAB22-sbvjU423.png)
查阅了很多大佬的资料，有许多不太清楚的地方已经补上了。
# REF
1. [RISC-V QEMU 的各种启动方式分析](https://www.bilibili.com/video/BV1hW4y1H7nS/)
2. [RISC-V UEFI 架构支持详解](https://tinylab.org/riscv-uefi-part1/)
3. [OpenSBI/U-Boot/UEFI 简介](https://zhuanlan.zhihu.com/p/512342633)
4. [QEMU 及 RISC-V 启动流程简介](https://www.cnblogs.com/YjmStr/p/16697848.html)
5. [uboot之fdt介绍](https://blog.csdn.net/u013165704/article/details/80374702)
6. [关于risc-v启动部分的思考](https://www.elecfans.com/d/1441031.html)
7. [risc-v与SBI与ABI](https://blog.csdn.net/u011011827/article/details/119185091)
8. [RISC-V OpenSBI 快速上手](https://zhuanlan.zhihu.com/p/509845102)
9. [Uboot与bootloader的关系](https://max.book118.com/html/2016/0710/47862461.shtm)
10. [RISC-V CPU加电执行流程](https://www.likecs.com/show-1023093.html)
11. [RISC-V64 opensbi启动过程](https://cloud.tencent.com/developer/article/1758282)
12. [opensbi下的riscv64裸机系列编程](https://www.elecfans.com/d/1445926.html)
13. [OpenSBI 汇编启动流程](https://blog.csdn.net/dai_xiangjun/article/details/123660424)
14. [d1哪吒开发板的启动流程分析](https://zhuanlan.zhihu.com/p/514269716)

