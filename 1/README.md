# 实验一（个人任务）

## 实验简介

本实验主要内容包括：

- 学习使用常用开发工具；
- 学习操作系统启动原理；
- 学习操作系统底层交互原理；
- 配置编译 Linux 内核；
- 在树莓派上完成 LED 实验；

完成本次实验你需要的硬件有：

- 一台计算机；
- 一个 Raspberry Pi 3 Model B+ （含电源）；
- 一张 2G 以上的 microSD 卡；
- 一个 microSD 读卡器；

注意：

- 请顺序阅读本实验指导文档**及文档中出现的超链接**，**完整阅读整个文档后再开展实验**；

- 实验需要提交的内容和评分规则见最后；

## 准备工作

- [安装 Linux 操作系统](./install)
- [熟悉 Linux 操作系统的常用命令](./linux)
- [学习使用 Bash](./bash)
- [学习使用 Git](./git)
- [熟悉 GitHub 并创建仓库](./github)
- [搭建树莓派开发环境](./pi)

注意：

- 准备工作中涉及到的工具、系统的使用方法在本实验中只需要较为基础的用法，不必完全精通；

- 本实验不强制要求使用 Linux 操作系统，使用 Windows/macOS/Windows Subsystem for Linux 等也可以顺利实验，但本实验指导文档主要针对 **Linux (Ubuntu 18.04.2)** 编写，对于其他操作系统不做详细指导，请同学们根据自己的实际情况调整；

## 实验内容

### 1-1 树莓派启动过程和原理

计算机从上电到用户可以操作的过程称为启动，这个过程比较复杂，被形象地称为 boot。

树莓派的启动分为多个阶段，请自行检索调研，也可以尝试在自己的树莓派上实际启动一个操作系统观察，**简述树莓派的启动过程和原理，记录在实验报告中**。

以下是一些可以参考的方向：

- 树莓派的启动分为哪几个阶段？
- 树莓派的启动和主流的计算机有何不同？
- 树莓派启动过程中至少用到了哪些文件系统？为什么？

在本地检查无误后，按照“实验提交材料”中的要求提交：

- 实验报告 `lab1-1.md` （或者 `lab1-1.pdf` 等）

### 1-2 利用 Linux 实现 LED 闪烁

这一部分实验中，你需要自己裁剪、编译、在树莓派上安装 Linux 内核（Kernel），并且借助这个 Kernel 实现 LED 闪烁。

详见：[为树莓派编译 Linux 内核](./kernel)

在本地检查无误后，按照“实验提交材料”中的要求提交：

- 实验报告 `lab1-2.md` （或者 `lab1-2.pdf` 等）；
- 编译好的内核文件（ `kernel7.img`）；

### 1-3 利用汇编实现 LED 闪烁

在上一部分实验中，我们借助 Linux Kernel 实现了 LED 闪烁，在本部分实验中，我们将使用汇编编写一个程序，替换 Linux Kernel，实现同样的效果。

详见：[汇编编写 LED 闪烁程序](./assembly)

在本地检查无误后，按照“实验提交材料”中的要求提交：

- 实验报告 `lab1-3.md` （或者 `lab1-3.pdf` 等）；
- 程序的汇编代码（ `led.s`）；

## 实验提交材料

请按照以下目录结构组织你的 GitHub 仓库：

```
.                       // Git 根目录
├── lab1                // 实验一根目录
│   ├── docs            // 实验一文档根目录
│   │   ├── lab1-1.md
│   │   ├── lab1-2.md
│   │   └── lab1-3.md   
│   └── files           // 实验一文件根目录
│       ├── kernel7.img // from lab1-2
│       └── led.s       // from lab1-3
└── README.md           // 整个仓库的 README 文件，可以将姓名学号写在其中
```

其中 `./lab1/files` 目录下的文件名必须严格遵守，否则可能导致实验检查失败。

## 评分规则和要点

本次实验满分共 10 分，其中：

- 正确创建 GitHub 仓库，并且正确添加权限（1 分）；
- 1-1 树莓派启动原理调研报告（3 分）；
- 1-2 中需要提交的文件检查通过，实验报告中要点齐全（共 3 分）；
- 1-3 中需要提交的文件检查通过，实验报告中要点齐全（共 3 分）；

注意：

- 实验报告除记录实验过程外，还需回答实验中要求回答的问题；
- 实验报告建议提交 Markdown 版本（如有图片一并打包提交） 或者 PDF 版本；
- **请注意合理控制实验报告字数**，节省时间！
