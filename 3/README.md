# Lab3（个人任务）

## 实验介绍

本实验通过 socket 编程，利用多进程、多线程技术，实现一个简易的支持并发的 HTTP 服务器，并可以利用线程复用， I/O 复用，缓存，预测等机制，提高 HTTP 服务器的性能。

### 启动一个简易 HTTP Server

python 自带了一个简单的 HTTP Server，我们可以通过以下命令来体验一个 HTTP Server 的工作：

```bash
mkdir -p /tmp/webroot && cd /tmp/webroot
echo "Hello from Python Server" > hello.html

# python2
python -m SimpleHTTPServer

# python3
python -m http.server
```

当屏幕中出现：

```bash
Serving HTTP on 0.0.0.0 port 8000 ...
```

即说明该 Server 已经成功运行，地址  `0.0.0.0`  不是一个具体的地址，它代表 Server 监听在本地所有的 IPv4 地址上， `port 8000` 代表 Server 监听在端口 `8000` 上。可以在浏览器打开本地回环地址 `127.0.0.1` （这也是本地 IPv4 地址之一），端口 `8000` 访问这个 Server。

在浏览器中打开以下地址，就可以看到来自 Python Server 的返回内容。

```bash
http://127.0.0.1:8000/hello.html
```

### HTTP 协议简介

接上一部分，HTTP 是一种基于文本的 TCP 协议，我们可以使用命令行工具 curl 以文本形式查看 HTTP 1.0 协议交互的全过程：

```bash
# curl --http1.0 -v http://127.0.0.1:8000/hello.html
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 8000 (#0)
> GET /hello.html HTTP/1.0
> Host: 127.0.0.1:8000
> User-Agent: curl/7.52.1
> Accept: */*
>
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Server: SimpleHTTP/0.6 Python/2.7.13
< Date: Tue, 14 May 2019 00:00:00 GMT
< Content-type: text/html
< Content-Length: 25
< Last-Modified: Mon, 13 May 2019 00:00:00 GMT
<
Hello from Python Server
* Curl_http_done: called premature == 0
* Closing connection 
```

其中 `>` 开头的为请求（request）内容，本实验中，我们可以忽略第三行、第四行，得到必要的请求内容如下：

```bash
> GET /hello.html HTTP/1.0
> Host: 127.0.0.1:8000
>
```

第一行是 HTTP 请求行，`GET` 代表 HTTP 方法，向指定的资源发出“显示”请求；`/hello.html` 为请求的资源路径；`HTTP/1.0` 为请求的协议版本。

接下来的行是请求行，`Host: 127.0.0.1:8000` 指明了请求的 Host，服务器的域名，以及服务器所监听的传输控制协议端口号。

注意，最后还有一个空行，这是不可省略的。

`<` 开头的为响应（response）内容，本实验中，我们只关注这些内容：

```
< HTTP/1.0 200 OK
< Content-Length: 25
<
Hello from Python Server
```

第一行为响应行，分别是协议版本和 HTTP 状态码，本实验中我们只考虑以下状态码：

```
200 OK 请求已成功，请求所希望的响应头或数据体将随此响应返回。

404 Not Found 请求失败，请求所希望得到的资源未被在服务器上发现。

500 Internal Server Error 本实验中未定义的各种错误。
```

接下来的行为响应头，其中 `Content-Length` 为回应消息体的长度，以字节为单位。

接着是一个空行，这是不可省略的。

空行之后，是返回的内容：`Hello from Python Server`，这就是我们请求的 `/hello.html` 的内容。

注意：

- 除了 `curl`，我们还可以使用 `nc 127.0.0.1 8000` 来手工构造请求测试。

- 现实生活中，HTTP 1.1 和 HTTP 2.0 协议使用更为广泛，关于 HTTP 1.1 协议的更多内容可以阅读参考资料中的 RFC 2616，也可以网上查找资料了解：`User-Agent`，`Accept` ，`Content-type` 等字段的含义和必要性。

### socket 简介

类似于本地进程间通过管道、消息队列、共享内存等通信，网络间的进程广泛使用 socket 进行通信，我们想要实现 HTTP server 就要实现一个 socket 服务端，接收来自客户端的请求，并根据客户端的的请求内容进行响应。

下面是本实验中可能用到的一些 socket 接口：

```
int socket(int af, int type, int protocol);
int bind(int sock, struct sockaddr *addr, socklen_t addrlen);
int listen(int sock, int backlog);
int accept(int sock, struct sockaddr *addr, socklen_t *addrlen);
```

请结合我们给出的一个有详细注释的 C 语言的例子掌握相应概念：

```c
// server.c
#include <math.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define BIND_IP_ADDR "127.0.0.1"
#define BIND_PORT 8000
#define MAX_RECV_LEN 1048576
#define MAX_SEND_LEN 1048576
#define MAX_PATH_LEN 1024
#define MAX_HOST_LEN 1024
#define MAX_CONN 20

#define HTTP_STATUS_200 "200 OK"

void parse_request(char* request, ssize_t req_len, char* path, ssize_t* path_len)
{
    char* req = request;

    // 一个粗糙的解析方法，可能有 BUG！
    // 获取第一个空格(s1)和第二个空格(s2)之间的内容，为 PATH
    ssize_t s1 = 0;
    while(s1 < req_len && req[s1] != ' ') s1++;
    ssize_t s2 = s1 + 1;
    while(s2 < req_len && req[s2] != ' ') s2++;

    memcpy(path, req + s1 + 1, (s2 - s1 - 1) * sizeof(char));
    path[s2 - s1 - 1] = '\0';
    *path_len = (s2 - s1 - 1);
}

void handle_clnt(int clnt_sock)
{
    // 一个粗糙的读取方法，可能有 BUG！
    // 读取客户端发送来的数据，并解析
    char* req_buf = (char*) malloc(MAX_RECV_LEN * sizeof(char));
    // 将 clnt_sock 作为一个文件描述符，读取最多 MAX_RECV_LEN 个字符
    // 但一次读取并不保证已经将整个请求读取完整
    ssize_t req_len = read(clnt_sock, req_buf, MAX_RECV_LEN);

    // 根据 HTTP 请求的内容，解析资源路径和 Host 头
    char* path = (char*) malloc(MAX_PATH_LEN * sizeof(char));
    ssize_t path_len;
    parse_request(req_buf, req_len, path, &path_len);
    
    // 构造要返回的数据
    // 这里没有去读取文件内容，而是以返回请求资源路径作为示例，并且永远返回 200
    // 注意，响应头部后需要有一个多余换行（\r\n\r\n），然后才是响应内容
    char* response = (char*) malloc(MAX_SEND_LEN * sizeof(char)) ;
    sprintf(response, 
        "HTTP/1.0 %s\r\nContent-Length: %zd\r\n\r\n%s", 
        HTTP_STATUS_200, path_len, path);
    size_t response_len = strlen(response);

    // 通过 clnt_sock 向客户端发送信息
    // 将 clnt_sock 作为文件描述符写内容
    write(clnt_sock, response, response_len);

    // 关闭客户端套接字
    close(clnt_sock);
    
    // 释放内存
    free(req_buf);
    free(path);
    free(response);
}

int main(){
    // 创建套接字，参数说明：
    //   AF_INET: 使用 IPv4
    //   SOCK_STREAM: 面向连接的数据传输方式
    //   IPPROTO_TCP: 使用 TCP 协议
    int serv_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    
    // 将套接字和指定的 IP、端口绑定
    //   用 0 填充 serv_addr （它是一个 sockaddr_in 结构体）
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    //   设置 IPv4
    //   设置 IP 地址
    //   设置端口
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = inet_addr(BIND_IP_ADDR);
    serv_addr.sin_port = htons(BIND_PORT);
    //   绑定
    bind(serv_sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr));

    // 使得 serv_sock 套接字进入监听状态，开始等待客户端发起请求
    listen(serv_sock, MAX_CONN);
    
    // 接收客户端请求，获得一个可以与客户端通信的新的生成的套接字 clnt_sock
    struct sockaddr_in clnt_addr;
    socklen_t clnt_addr_size = sizeof(clnt_addr);

    while (1) // 一直循环
    {
        // 当没有客户端连接时， accept() 会阻塞程序执行，直到有客户端连接进来
        int clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_addr, &clnt_addr_size);
        // 处理客户端的请求
        handle_clnt(clnt_sock);
    }

    // 实际上这里的代码不可到达，可以在 while 循环中收到 SIGINT 信号时主动 break
    // 关闭套接字
    close(serv_sock);
    return 0;
}
```

编译运行方法：

```
gcc server.c -o server
./server
```

测试方法（新开一个终端）：

```bash
curl http://127.0.0.1:8000/hello
```

如果成功，将会收到内容为 `/hello` 的 HTTP 响应。

打开 `-v` (verbose) 选项，可以看到服务器返回的全部内容：

```bash
curl http://127.0.0.1:8000/hello -v
```

示例程序中使用到的各调用的具体用法请同学们使用 `man` 命令查阅手册，如：`man 2 listen`。

下面是一些本实验中需要重点理解的概念：

#### 请求队列

当套接字正在处理客户端请求时，如果有新的请求进来，套接字是没法处理的，只能把它放进缓冲区，待当前请求处理完毕后，再从缓冲区中读取出来处理。如果不断有新的请求进来，它们就按照先后顺序在缓冲区中排队，直到缓冲区满。这个缓冲区，就称为请求队列（Request Queue）。

缓冲区的长度（能存放多少个客户端请求）可以通过 listen() 函数的 backlog 参数指定，但究竟为多少并没有什么标准，可以根据你的需求来定。如果将 backlog 的值设置为 SOMAXCONN，就由系统来决定请求队列长度，这个值一般比较大，可能是几百，或者更多。

当请求队列满时，就不再接收新的请求，对于 Linux，客户端会收到 ECONNREFUSED 错误，为了使得我们的服务器可用性尽可能高，这是我们要避免的。

注意：listen() 只是让套接字处于监听状态，并没有接收请求。接收请求需要使用 accept() 函数。

#### 数据的接收和发送

Linux 不区分文件描述符和套接字，也就是说，你可以使用 Lab2 中对于文件描述符中的各种操作完成对套接字的操作。

可能会用到的接口：

```
ssize_t write(int fd, const void *buf, size_t nbytes);
ssize_t read(int fd, void *buf, size_t nbytes);
```

注意：这两个接口使用时很容易出错，如 read 时，不能假定能一次读取完 HTTP 请求的全部内容，而应该根据 HTTP 协议，不断尝试读取，直到读取到两个换行符（`\r\n\r\n`）时才算读取完成。

具体用法请同学们自行查阅手册，掌握各参数及返回值的意义。

## 实验要求

在 socket 简介一节，我们给出了一个简易的 HTTP Server 原型，它能够一个接一个的处理请求，当 `handle_clnt` 正在运行时，新的请求将被放置于请求缓冲区，在当前请求处理完成后，才会进行下一个请求的处理。

另外，这个版本的 HTTP 服务器存在以下问题：

- 没有实现并发处理，当客户端发送数据过慢时，新的请求等待时间过长；
- 没有很好的解析和检验 HTTP 头；
- 没有实现读取请求资源内容的功能；
- 没有实现返回不同的 HTTP 状态码；
- 没有实现错误和异常处理；

为了解决以上问题，本实验要求同学们在理解示例代码的基础上，实现一个高性能 HTTP Server，能尽可能准确、快速处理并发的请求。

### 多进程和多线程

示例代码中，请求是顺序阻塞进行解析响应，这极大拖慢了 Server 的响应速度，我们可以将处理响应的部分移至新线程或新进程中处理，如：`fork()` 创建新进程来处理请求，也可以用 `pthread` 来创建新线程，这样能大大提高 Server 的并发性能，在多个请求到达时也能快速响应。

请同学们自行选择一种方案，说明理由，并用代码实现。

### 解析和检验 HTTP 头

目前 `parse_request()` 只能处理正确的请求，对于畸形的请求，可能会造成严重问题，请根据 HTTP 协议简介一节完善这一部分的代码，使得对 HTTP 请求的检查更健壮，对于畸形的请求，可以返回 `500 Internal Server Error`。

### 实现读取请求资源内容

真正的 HTTP Server 会尝试返回用户请求的资源内容（文件内容），请参考 Python Server 的行为，实现这一部分内容。

例如：

- 请求的资源路径是 `/index.html` 则尝试返回当前运行目录下的 `index.html` 文件的内容；
- 请求的资源路径是一个目录，则返回 500 错误；

注意：

- 资源文件根目录为程序运行时的当前目录；
- 返回头仅要求完成  `Content-Length`；
- 当请求的资源（文件）不存在时，可以返回 `404 Not Found`；
- 当出现其他错误时，可以返回 `500 Internal Server Error`；

### 实现错误和异常处理

示例代码中没有对各种错误和异常做良好的处理，请根据各个调用可能出错的情况进行分析，使得 Server 尽可能健壮，不会运行时崩溃，提高 Server 的可用性。

### 使用进程池/线程池机制（选做）

每次创建新线程/新进程需要额外的开销，请自行思考设计，是否可以预先创建一组线程/进程，在请求到达时分配给其中一个，以减少创建的额外开销。

### 使用高级 I/O 模式（选做）

Linux 提供 `select` `epoll` 等高级的 I/O 模式，请自行了解相关概念，使用 `select` `epoll` 等提高服务器并发性能，完成本项的同学可以使用 siege 测试并记录性能指标。

### 使用缓存机制（选做）

对于经常访问的资源，可以使用内存缓存或预取对资源读取进行加速。一个常见的例子是，html 文件中包含的图片，Javascript 代码，CSS 代码等具有显著的访问模式，可以设计机制对这类特定的访问顺序进行缓存或预取，使得响应速度更快。

注意：Linux 本身对文件有透明的缓存机制，如果要自己实现，请注意比较原版和自己实现的性能指标差异。

### 未提及的细节问题

对于本文档中未提及的细节问题，请先浏览 (FAQ)[https://osh-2019.github.io/faq/#lab3] 查看是否有相关说明，对于没有说明的问题，可以询问助教，也可以做出合理的假设，并以此为基础做出设计。注意：本次实验重点为提高性能指标。

## 测试说明

### 对程序要求

编程语言：C/C++，使用其他语言（如 Rust）请和助教联系确认，核心功能部分的代码不能使用库。

监听 IP：`0.0.0.0`

监听端口：`8000`

文件根目录：程序运行时的当前目录

### 测试环境

- Web Server 运行于树莓派 3B+ (Raspbian GNU/Linux 9, Linux raspberrypi 4.14.98-v7+)
- 测试机器在树莓派同一局域网下另一台 Linux 机器（同学们在测试时注意关闭树莓派的各类防火墙）

### 测试指标

测试由正确性测试和性能测试组成。

#### 正确性测试（40%）

- 主要考察指标：
  - 是否可以正确响应
  - 返回头部是否正确
  - 返回内容是否正确

注意：正确性测试过程中，可能会有文件的删改。

#### 性能测试（60%）

- 使用 benchmark 工具 [Siege](https://github.com/JoeDog/siege) (v4.0.4) 测试；
- 例子：`siege -c 50 -r 10 http://127.0.0.1:8000/index.html`
- 主要考察指标：
  - Availability
  - Throughput
  - Concurrency

### 关于 URL 规范

HTTP 中的 URL 有着非常复杂而详细的定义，可以通过查阅参考资料中的 RFC 1738 了解。本实验中我们不要求（也不推荐）去完善实现 URL 规范（不是本实验的重点）。我们的测试仅包含最常见的情况，并保证只包含大小写英文字母，数字，连字符下划线和斜线，如：

- http://127.0.0.1/dir/index0.html
- http://127.0.0.1/.a/
- http://127.0.0.1/b/
- http://127.0.0.1/ind.html
- http://127.0.0.1/ind.html/

## 实验提交材料

请按照以下目录结构组织你的 GitHub 仓库：

```
.                       // Git 根目录
├── lab3                // 实验三根目录
│   ├── docs            // 实验三文档根目录
│   │   └── lab3.md     // 实验三实验报告
│   └── files           // 实验三文件根目录
│       ├── Makefile    // Makefile 或其他文件
│       └── server.c    // 源代码
└── ...                 // 其他文件
```

实验报告中请注明编译的方法和运行方法，使得 `./lab1/files` 目录下源代码在测试环境中编译出文件名为 `server` 的二进制可执行文件，否则可能导致测试失败。

## 评分规则和要点

本次实验满分共 10 分，其中：

- 实验报告共 2 分；
  - 编译和运行方法说明；
  - 描述整体设计，使用的技术；
- 正确性和性能测试共 8 分；

注意：

- 实验报告建议提交 Markdown 版本（如有图片一并打包提交） 或者 PDF 版本；
- **请注意合理控制实验报告字数**！

## 参考资料

- RFC 2616 https://tools.ietf.org/html/rfc2616
- RFC 1738 https://tools.ietf.org/html/rfc1738
- socket(2) http://man7.org/linux/man-pages/man2/socket.2.html
- Unix Socket Tutorial https://www.tutorialspoint.com/unix_sockets/
