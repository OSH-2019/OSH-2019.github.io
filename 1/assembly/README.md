# 利用汇编实现 LED 闪烁

## 检查树莓派开发环境

根据准备工作中的“搭建树莓派开发环境”一节，检查自己是否已经准备好开发环境，请确保没有问题后继续。

## 汇编基础

请根据参考资料中"ARM 汇编基础"的四个链接学习 ARM 汇编的基础知识，本部分实验中，我们要用到的指令有（但不限制于此）：

```assembly
ldr
str

lsl
add
and

cmp

b
bne
```

## 编写汇编程序

首先我们通过以下汇编代码点亮 LED（这也可以检查你的开发环境是否配置好）：

```assembly
.section .init
.global _start
_start:

ldr r0, =0x3F200000

mov r1, #1
lsl r1, #27
str r1, [r0, #8]

mov r1, #1
lsl r1, #29
str r1, [r0, #28]
```

这段汇编代码不算太长，其中：

```assembly
ldr r0, =0x3F200000
```

这句指令的含义是将 `0x3F200000` 载入到 `r0` 寄存器中，这是树莓派上 GPIO 控制器的内存地址，我们向这个内存周围写入一些数据即可完成对树莓派板载端口的控制，进而控制绿色的 ACT LED 灯。注意：这个地址（以及下面出现的所有 Magic Number）并非只能由助教告诉你，你也可以自己从手册（见参考资料）中查到。

```assembly
mov r1, #1
lsl r1, #27
```

这是构造一个我们即将使用的数据，两条指令最终在 `r1` 中留下一个二进制为 `0b1000000000000000000000000000` 的值。

```assembly
str r1, [r0, #8]
```

这是一个存储指令，代表把`r1` 的值存入 `r0`（还记得吗，是 GPIO 控制器的地址）偏移为 `0x08` 的地方。

GPIO 中 00-24 字节偏移是功能选择区，其中每 4 个字节负责控制 10 个端口（即：前 4 字节，控制 1～10 端口，接下来的 4 字节控制 11～20 端口，以此类推），我们需要控制的 ACT 是 GPIO29，因此我们需要先跳过前 20 个端口，即  `(20/10)*4 = 8` 个字节，这是 `0x08` 偏移的来历，然后我们需要跳过 9 个端口，每个端口有 3bit，所以我们要控制的是第 `3 * 9 = 27` 个 bit，这是 `0b1000000000000000000000000000` 的来历，通过这一条指令，我们成功将 GPIO29（ACT LED）设置为了输出模式，以备之后点亮 LED。

```assembly
mov r1, #1
lsl r1, #29
str r1, [r0, #28]
```

类似的道理，`r1` 被我们设为了 `0b100000000000000000000000000000`，并且存入了 GPIO 控制器偏移 `28` 的位置。这是因为 GPIO 28-36 这 8 个字节中每个 bit 分别对应一个端口，将某一 bit 设置为 1，则会将对应的端口打开（前提是已经在上一步中设置好了输出模式）。

我们的这三条指令实现了将 GPIO29 端口打开，即点亮了 LED。

**注意：请确保你已经明白上面这段汇编中每一条语句的意思，然后再继续下面的操作。**

将这段汇编代码保存为 `test.s` ，并且使用以下命令汇编：

```shell
arm-linux-gnueabihf-as test.s -o test.o
arm-linux-gnueabihf-ld test.o -o test.elf
arm-linux-gnueabihf-objcopy test.elf -O binary test.img
```

## 安装内核

首先完成“为树莓派编译 Linux 内核”中的烧录磁盘镜像（[lab1.img](https://github.com/OSH-2019/OSH-2019.github.io/raw/master/1/resources/lab1.img)）的工作，但是不需要将 Linux Kernel 复制到 SD 卡的第一个分区（如：`/dev/sdc1` 中），而是将你编写的 led 程序放入其中。

使用相似的命令，将 `test.img` 装载到 SD 卡中：

```bash
mkdir mnt
mkdir mnt/fat32
sudo mount /dev/sdc1 mnt/fat32
sudo cp test.img mnt/fat32/kernel7.img # 装载 test 程序
sudo umount mnt/fat32
```

完成之后，可以拔下 SD 卡，插入树莓派实验，如果一切正常，你将会在几秒钟的短暂启动后观察到绿色 LED 常亮。至此，你实现了完全不依靠任何操作系统（如：Linux）与树莓派的硬件进行交互！

## 编写更复杂的功能

目前我们编写了一个仅有 10 行的汇编小程序，实现了点亮树莓派上的绿色 LED 灯，下面我们希望在 `test.s` 的基础上编写一个和上一部分实验中 `init` 一样能让 LED blink 的小程序。

为了编写新的程序，你可能需要以下提示：

- GPIO 40-48 这 8 个字节偏移中每个 bit 分别对应一个端口，将某一 bit 设置为 1，则会将对应的端口关闭。

- `b` 指令能实现无条件跳转，以实现 C 语言中类似于循环的控制流控制，如：

  ```assembly
  loop:
  @ some code ...
  b loop
  ```

  这将会一直重复运行中间的代码区域，除非有指令主动跳出这一循环。

  而以 `b` 为前缀的其他一些指令能完成有条件跳转，如 `bne`：

  ```assembly
  cmp r1, #0         @ 将 r1 和 0 比较（这会影响标志寄存器）
  bne loop
  ```

  则是在 ne (Not Equal) 时跳转到 `loop` 标签。

- 内存地址 `0x3F003000` 处有一个计时器控制器，在偏移 4 字节处，你可以读取到一个 1MHz 自增的循环计数值（这个值是 8 字节的，我们可以只使用低 4 字节），如果你需要精确控制时间，它可能很有用，一个使用的例子如下：

  ```assembly
  ldr r2, =0x3F003000
  ldr r3, [r2, #4]   @ 读取计时器的低 4 字节
  @ maybe check the value of r3?
  @ other code
  ```
  
  请自行查阅参考资料中的手册，得知 System Timer 的计数频率。

将新的汇编代码保存为 `led.s`，并且使用类似的命令汇编、装载。

```bash
arm-linux-gnueabihf-as led.s -o led.o
arm-linux-gnueabihf-ld led.o -o led.elf
arm-linux-gnueabihf-objcopy led.elf -O binary led.img
sudo mount /dev/sdc1 mnt/fat32
sudo cp led.img mnt/fat32/kernel7.img # 装载 LED Blink 程序
sudo umount mnt/fat32
```

完成之后将 SD 卡插入树莓派中启动，我们希望能在树莓派上看到和上一部分实验中相似的闪烁效果（闪烁间隔 2s，不要求最后停止）。

## 问题探究

完成实验后，你可以从以下问题中选取一些进行探究：

（假设 SD 卡的第一个分区为 `/dev/sdc`，第二个分区为 `/dev/sdc2`）

- 在这一部分实验中 `/dev/sdc1` 中除了 `kernel7.img` （即 `led.img`）以外的文件哪些是重要的？他们的作用是什么？
- 在这一部分实验中 `/dev/sdc2` 是否被用到？为什么？
- 生成 `led.img` 的过程中用到了 `as`, `ld`, `objcopy` 这三个工具，他们分别有什么作用，我们平时编译程序会用到其中的哪些？

## 参考资料

ARM 汇编基础：

- https://azeria-labs.com/writing-arm-assembly-part-1/

- https://azeria-labs.com/arm-data-types-and-registers-part-2/

- https://azeria-labs.com/arm-instruction-set-part-3/

- https://azeria-labs.com/memory-instructions-load-and-store-part-4/

BCM2837 ARM Peripherals：https://cs140e.sergio.bz/docs/BCM2837-ARM-Peripherals.pdf
