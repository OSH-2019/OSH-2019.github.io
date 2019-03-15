# 搭建树莓派开发环境

由于树莓派（ARM 架构）和绝大多数人的计算机（x86 架构）不同，所以我们需要使用一套特殊的工具来完成交叉编译。

## 准备工具链

对于一般的需求，我们可以直接使用 GitHub 仓库中 Raspberry Pi 预编译好的工具链：

```shell
cd ~
mkdir lab
cd lab
git clone https://github.com/raspberrypi/tools ~/lab/tools
```

然后将工具链二进制文件夹的路径写入环境变量 PATH 中，以方便之后的使用，以 bash 为例：

```shell
echo PATH=\$PATH:~/lab/tools/arm-bcm2708/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin >> ~/.bashrc
source ~/.bashrc
```

完成以上步骤之后，尝试运行：
```shell
arm-linux-gnueabihf-gcc --version
```

输出应该类似于：
```
arm-linux-gnueabihf-gcc (crosstool-NG linaro-1.13.1+bzr2650 - Linaro GCC 2014.03) 4.8.3 20140303 (prerelease)
Copyright (C) 2013 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

这说明安装成功，如果没有出现类似的输出，说明以上安装步骤出现了问题，请检查 `~/.bashrc` 中 PATH 中的路径是否正确。



注意：

- 本文档主要针对 **Linux (Ubuntu 18.04.2)** 编写，其他系统请自行调整；
- 请大家确保这一步正确完成，再进行后续的操作；
- 除了下载预编译好的工具链，也可以自行编译 GNU 工具；



## 参考资料

https://www.raspberrypi.org/documentation/linux/kernel/building.md