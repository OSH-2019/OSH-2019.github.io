# 为树莓派编译 Linux 内核

## 下载内核源代码

首先需要准备好 git 工具，git 可以从各发行版包管理器很方便的安装，如：

```bash
sudo apt install git
```

然后开始 clone 内核源代码：

```bash
cd $HOME
mkdir $HOME/lab/
cd $HOME/lab
git clone --depth=1 --branch rpi-4.14.y https://github.com/raspberrypi/linux
```

成功 clone 之后，你应该会看到类似的输出：

```bash
Cloning into 'linux'...
remote: Enumerating objects: 65818, done.
remote: Counting objects: 100% (65818/65818), done.
remote: Compressing objects: 100% (59147/59147), done.
remote: Total 65818 (delta 6921), reused 19903 (delta 5698), pack-reused 0
Receiving objects: 100% (65818/65818), 174.49 MiB | 9.67 MiB/s, done.
Resolving deltas: 100% (6921/6921), done.
Checking connectivity... done.
Checking out files: 100% (61885/61885), done.
```

## 检查树莓派开发环境

根据准备工作中的“搭建树莓派开发环境”一节，检查自己是否已经准备好开发环境，请确保没有问题后继续。

## 准备编译所需工具

在正式开始编译前，还需要安装这些工具：

```bash
sudo apt install bc bison flex libssl-dev libncurses5-dev
```

## 配置内核编译选项

进入源代码目录，开始编译：

```bash
cd $HOME/lab/linux
KERNEL=kernel7
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2709_defconfig # A
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig        # B
```

其中 A 语句的含义是使用 `bcm2709_defconfig` 默认配置（适用于 3B+），B 语句将会基于之前的 `.config` 显示 Linux Kernel 的配置菜单，其中有各种各样的选项，请同学们根据本实验所需，酌情关闭一些多余的特性或功能，使得我们编译出来的内核尽可能精简，请将你的**修改及理由**记录到实验报告中。**注意，如果你没有足够的理由判断一个选项应该关闭，则最好不要关闭它，以防不能正常启动**。

在下面的步骤中，我们没有在 `bcm2709_defconfig` 的基础上关闭任何多余选项（也就是说，我们只运行了 A 语句，没有运行 B 语句），如果你是第一次编译内核，为了降低出错的可能性，我们建议你也这样做。

## 开始编译！

这一条命令将会花费较长的时间，可以根据自己的机器添加 -j (jobs) 参数，如：

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j8 zImage modules dtbs
```

其中 8 是你的机器核数，可以通过 `grep -c '^processor' /proc/cpuinfo` 命令查看。

根据你的编译选项配置和机器配置，这一编译过程将持续 5～40 分钟不等，这段时间你可以离开电脑（但不要拔掉电源）休息，也可以玩一会儿手机。

如果编译过程顺利结束，最后将会没有报错信息。

## 安装内核

安装内核前，请确保你已经将 SD 卡通过读卡器连接到了自己的计算机上，如果是在虚拟机中使用，请确保读卡器被直接连接到了虚拟机内。

执行：

```
lsblk
```

应该类似于这样的信息输出：

```
sda      8:0    0    20G  0 disk
└─sda1   8:1    0    20G  0 part /
sdc      8:32   1 119.1G  0 disk
├─sdc1   8:33   1  43.9M  0 part
└─sdc2   8:34   1    20M  0 part
sr0     11:0    1  1024M  0 rom
```

其中，大小约为 128GB 的 sdc （设备路径是 `/dev/sdc`）是我们的 SD 卡，而其他的可能是计算机本地的其他设备，如果在你的计算机上有多块硬盘、U 盘，请在执行下面的命令时格外小心，将 `sdc` 替换为实际的设备名称，以防止丢失本地数据。

**注意：如果你的 SD 卡上有重要数据，请自行备份，这些数据将在后面的步骤被覆盖。**

在这一步中，先确保没有挂载 SD 卡，没有正在使用 SD 的程序。

```
sudo mount | grep /dev/sdc
```

如果这条命令有输出，则需要手动 `umount` 。

然后使用我们提供好的磁盘镜像（[lab1.img](https://github.com/OSH-2019/OSH-2019.github.io/raw/master/1/resources/lab1.img)），以避免自行分区的工作，这个磁盘镜像只缺少你即将安装的内核（`kernel7.img`），即可自动运行助教写好的 init 程序。

```bash
wget https://github.com/OSH-2019/OSH-2019.github.io/raw/master/1/resources/lab1.img
sudo dd if=lab1.img of=/dev/sdc bs=2M status=progress
```

`dd` 执行完之后，请检查  `lab1.img`  是否成功刻入 SD 卡中，可以使用 `fdisk` 工具来检查：

```bash
sudo fdisk -l /dev/sdc
```

输出应该和以下非常类似：

```
Disk /dev/sdc: 119.1 GiB, 127865454592 bytes, 249737216 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x7ee80803

Device     Boot Start    End Sectors  Size Id Type
/dev/sdc1        8192  98045   89854 43.9M  c W95 FAT32 (LBA)
/dev/sdc2       98304 118783   20480   10M 83 Linux
```

检查之后，将编译好的内核复制到 SD 卡中：

```bash
mkdir mnt
mkdir mnt/fat32
sudo mount /dev/sdc1 mnt/fat32

KERNEL=kernel7
sudo cp arch/arm/boot/zImage mnt/fat32/$KERNEL.img # 安装内核
sudo umount mnt/fat32
```

至此内核就已经安装完成了，可以拔下 SD 卡到树莓派中测试。

## 最终测试

我们已经提前在 `lab1.img` 中准备好了一个`init` 程序，当 Linux Kernel 启动后，根据内核的参数，`init` 程序会自动运行，其功能为：在屏幕上打印 `Hello OSH-2019`，同时以 2 秒一次的速度闪烁树莓派上的绿色 LED 灯，开机 2 分钟后，程序自动退出，Linux Kernel 随之 panic，LED 之后将熄灭，不再闪烁。

如果你将 SD 卡插入后成功观测到以上现象（如果没有外接屏幕，只需要观察绿色 LED 灯的闪烁即可），则说明你编译的 Linux 内核已经顺利在树莓派上启动起来了！

## 问题探究

完成实验后，你可以从以下问题中选取一些进行探究：

- `/dev/sdc1` 中除了 `kernel7.img` 以外的文件哪些是重要的？他们的作用是什么？
- `/dev/sdc1` 中用到了什么文件系统，为什么？可以换成其他的吗？
- `/dev/sdc1` 中的 kernel 启动之后为什么会加载 `/dev/sdc2` 中的 `init` 程序？
- `/dev/sdc2` 中的 `init` 正常工作至少需要打开 Linux Kernel 的哪些编译选项？
- 开机两分钟后，`init` 程序退出， Linux Kernel 为什么会 panic？

```
提示：init 程序是 C 语言静态编译得到，使用 printf() 向屏幕输出内容，使用 sleep()  完成延时，使用 /sys/class/leds/led0/brightness 这个文件控制 LED 的闪烁，还有可能用到 /proc， /dev 下的一些特殊文件。程序在完成 60 次闪烁后，结束运行，返回 0。
```

## 常见错误

- kernel 安装错误或者没有安装成功（不存在）：绿灯连续闪烁 7 下，停止 2 秒，继续闪烁，外接屏幕观察到彩虹色图；
- 磁盘镜像刻录失败（dd 失败）或 SD 卡损坏：绿色 LED 灯不闪烁，外界屏幕无彩虹色图，也无其他任何输出；
- 测试时没有观察到成功的现象很可能是因为裁剪内核时破坏了 init 运行所必须要的选项；
