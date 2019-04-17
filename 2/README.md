
实验二（个人任务）
==================

# 实验目的

为了让用户可以控制系统，Linux系统一般会运行一个shell程序，比如bash、zsh等。本实验旨在构建出一个精简的shell进程，接受用户控制。

# 编写shell程序

首先，大家可以将本页底部助教编写的一个示例程序命名为`shell.c`。然后在该目录下执行：

```Shell
gcc -static -o sh shell.c
```

这会编译出名为`sh`的可执行程序。然后输入 ./sh 运行，应当就能使用这个非常简陋的shell程序了——shell程序提示你输入命令。你可以输入`pwd`来查看当前所在的目录，或者输入`cd 某目录`来改变所在的目录，也可以调用任何busybox中提供的程序，如`ls`、`cat`等等。

建议你先完全理解这个示例程序的工作原理、查清楚里面用到的各个函数的功能。

*思考一下：为什么cd命令必须做成shell程序的功能，而不能编译为外部程序，通过调用来运行呢？*

这个示例shell非常简陋，你的任务是在以下几个方面对它进行升级：

## 更健壮（推荐做，但不要求）

目前的示例程序非常脆弱，无法处理不良的输入（如`cd /`可以运行，但将中间的一个空格改成两个就不行），也不检查各种可能的错误（`chdir`、`fork`、`exec`等调用都可能出错）。建议你将它改得更健壮，以方便之后的进一步开发和调试。

## 支持管道

形如`env | wc`这样的命令利用了“管道”语法，将两条不同的命令对接在一起同时运行。`|`的意思是将前面的命令`env`（输出所有环境变量）的输出，作为`wc`（统计行数）的输入（这样就能统计出环境变量的总数）。请你观察并学习这个语法的效果，为你的shell程序实现这一功能。

你可能要用到的函数：`pipe`、`close`、`dup`。你可以执行`man 函数名`来查看系统自带的文档，或者上网搜索更多信息。

## 支持修改环境变量

许多程序和系统功能都依赖于环境变量，比如当你执行`ls`命令时，系统就是通过查询`PATH`环境变量来确定`ls`这个程序到底在哪里的。你可以运行`env`命令来查看当前的所有环境变量，也可以通过`export MY_OWN_VAR=abc`这样的命令来修改环境变量。请为你的shell程序实现这一功能。

你可能要用到的函数：`getenv`、`setenv`。

## 支持基本的文件重定向

形如 ls > out.txt 会将 ls 命令的输出重定向到 out.txt 文件中，具体地说，会将 out.txt 关联到文件描述符 STDOUT_FILENO ，然后再运行相应的命令。

类似的， ls >> out.txt 会将输出追加（而不是覆盖）到 out.txt 文件， cat < in.txt 会将程序的标准输入重定向为文件 in.txt。

请为你的shell程序实现 > 、 >> 和 < 的功能。

你可能要用到的函数：`open`、`close`、`dup`。

## 支持基于文件描述符的文件重定向、文件重定向组合（选做）

形如 cmd 10> out.txt 和 cmd 20< in.txt 以及 cmd 10>&20 30< in.txt 这样的命令会将打开文件描述符10、20和30并重定向到相应的文件。请自行查找资料，实现这些文件重定向。

```Shell
cmd << EOF
this
output
EOF
```
上述程序会将"this\noutput\n"作为标准输入重定向给 cmd。请实现这种重定向方式。

cmd <<< text 会将"text\n"作为标准输入重定向给 cmd。请事先这种重定向方式。

## 更多功能（选做）

我们一般使用的shell非常强大，你还可以自行了解下面这些语法的含义：

```Shell
echo $PATH
A=1 env
alias ll='ls -l'
echo ~
echo ~root
(sleep 10; echo aha) &
if true; then ls; fi
```

## 示例程序

```C
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>
#include <sys/types.h>

int main() {
    /* 输入的命令行 */
    char cmd[256];
    /* 命令行拆解成的各部分，以空指针结尾 */
    char *args[128];
    while (1) {
        /* 提示符 */
        printf("# ");
        fflush(stdin);
        fgets(cmd, 256, stdin);
        /* 清理结尾的换行符 */
        int i;
        for (i = 0; cmd[i] != '\n'; i++)
            ;
        cmd[i] = '\0';
        /* 拆解命令行 */
        args[0] = cmd;
        for (i = 0; *args[i]; i++)
            for (args[i+1] = args[i] + 1; *args[i+1]; args[i+1]++)
                if (*args[i+1] == ' ') {
                    *args[i+1] = '\0';
                    args[i+1]++;
                    break;
                }
        args[i] = NULL;

        /* 没有输入命令 */
        if (!args[0])
            continue;

        /* 内建命令 */
        if (strcmp(args[0], "cd") == 0) {
            if (args[1])
                chdir(args[1]);
            continue;
        }
        if (strcmp(args[0], "pwd") == 0) {
            char wd[4096];
            puts(getcwd(wd, 4096));
            continue;
        }
        if (strcmp(args[0], "exit") == 0)
            return 0;

        /* 外部命令 */
        pid_t pid = fork();
        if (pid == 0) {
            /* 子进程 */
            execvp(args[0], args);
            /* execvp失败 */
            return 255;
        }
        /* 父进程 */
        wait(NULL);
    }
}
```

## 实验提交材料

请按照以下目录结构组织你的 GitHub 仓库：

```
.                       // Git 根目录
├── lab1                // 实验一根目录
├── lab2                // 实验二根目录
│   ├── src               // 实验二代码目录
│   │   ├── shell.c         // 源代码
│   │   ├── other.c         // 其他代码文件（可选）
│   │   └── ...   
│   ├── Makefile          // 如果实验代码为多个文件（而非单一的shell.c），请包含Makefile文件
│   └── README.md         // 实验二实验报告
└── README.md           // 整个仓库的 README 文件，可以将姓名学号写在其中
```

仓库`lab2`目录中需要有`src/shell.c`文件或`Makefile`文件（你编写的shell源码），以及一个README.md文件（实验报告）。

注：Makefile文件应该能在`lab2`目录下编译生成一个`sh`可执行程序。

你的shell应当能从键盘输入类似下方列出的命令并成功运行。准确地说，你所编写的shell程序应当支持`cd`、`pwd`这两个内建命令，支持修改环境变量，支持文件重定向，支持所有系统命令，支持管道符号`|`。

```
cd /
pwd
ls
ls | wc
ls | cat | wc
env
export MY_OWN_VAR=1
env | grep MY_OWN_VAR
```

*程序检测到的抄袭将会进行人工复核，确认的抄袭行为将导致本次实验0分，请不要复制他人的代码或提交非本人编译的结果。*
