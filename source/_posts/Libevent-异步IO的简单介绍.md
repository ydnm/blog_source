---
title: 'Libevent: 异步IO的简单介绍'
date: 2017-08-09 03:22:02
categories: 网络编程
tags: Libevent
---

## Libevent: 异步IO的简单介绍

```
These documents are Copyright (c) 2009-2012 by Nick Mathewson, and are made available under the Creative Commons Attribution-Noncommercial-Share Alike license, version 3.0. Future versions may be made available under a less restrictive license.

Additionally, the source code examples in these documents are also licensed under the so-called "3-Clause" or "Modified" BSD license. See the license_bsd file distributed with these documents for the full terms.

For the latest version of this document, see http://www.wangafu.net/~nickm/libevent-book/TOC.html

To get the source for the latest version of this document, install git and run "git clone git://github.com/nmathewson/libevent-book.git"
```

> 原文地址： http://www.wangafu.net/~nickm/libevent-book/01_intro.html

大部分初学者学习网络都是从阻塞IO调用开始的。一个IO调用如果是同步的，当我们调用它，它会阻塞直到操作完成，或者在尝试足够多次后你的网络栈放弃继续尝试。比如如果你用Tcp方式调用connect()操作，你的操作系统会发一个SYN包到目标主机端口。你的应用会阻塞在connect()直到你收到目标主机发来的SYN ACK包或者尝试连接多次后放弃继续连接。

下面有一个简单的客户端使用阻塞调用的例子。它尝试连接谷歌网站www.google.com, 并发送一段HTTP请求， 然后在终端打印对方的响应。

> 例1： 一个简单的HTTP客户端

```
/* For sockaddr_in */
#include <netinet/in.h>
/* For socket functions */
#include <sys/socket.h>
/* For gethostbyname */
#include <netdb.h>

#include <unistd.h>
#include <string.h>
#include <stdio.h>

int main(int c, char **v)
{
    const char query[] =
        "GET / HTTP/1.0\r\n"
        "Host: www.google.com\r\n"
        "\r\n";
    const char hostname[] = "www.google.com";
    struct sockaddr_in sin;
    struct hostent *h;
    const char *cp;
    int fd;
    ssize_t n_written, remaining;
    char buf[1024];

    /* Look up the IP address for the hostname.   Watch out; this isn't
       threadsafe on most platforms. */
    h = gethostbyname(hostname);
    if (!h) {
        fprintf(stderr, "Couldn't lookup %s: %s", hostname, hstrerror(h_errno));
        return 1;
    }
    if (h->h_addrtype != AF_INET) {
        fprintf(stderr, "No ipv6 support, sorry.");
        return 1;
    }

    /* Allocate a new socket */
    fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) {
        perror("socket");
        return 1;
    }

    /* Connect to the remote host. */
    sin.sin_family = AF_INET;
    sin.sin_port = htons(80);
    sin.sin_addr = *(struct in_addr*)h->h_addr;
    if (connect(fd, (struct sockaddr*) &sin, sizeof(sin))) {
        perror("connect");
        close(fd);
        return 1;
    }

    /* Write the query. */
    /* XXX Can send succeed partially? */
    cp = query;
    remaining = strlen(query);
    while (remaining) {
      n_written = send(fd, cp, remaining, 0);
      if (n_written <= 0) {
        perror("send");
        return 1;
      }
      remaining -= n_written;
      cp += n_written;
    }

    /* Get an answer back. */
    while (1) {
        ssize_t result = recv(fd, buf, sizeof(buf), 0);
        if (result == 0) {
            break;
        } else if (result < 0) {
            perror("recv");
            close(fd);
            return 1;
        }
        fwrite(buf, 1, result, stdout);
    }

    close(fd);
    return 0;
}
```

这段代码中所以的网络API调用都是阻塞的。`gethostbyname`调用会阻塞直到执行成功或者解析`www.google.com`失败。`connect`调用会阻塞直到连上。`recv`调用会阻塞直到接收到足够的数据或者连接关闭。`send`调用会阻塞直到向内核写缓存里写了部分数据。

这样看来，使用阻塞IO也未尝不可。如果你的程序在调用阻塞调用的时候不需要去处理其他的事情的话，阻塞IO很适合。但是如果你期望你的程序能同时处理多个连接，具体一点就是假如我们前面的例子中，你需要从2个连接中读取数据，并且你不清楚哪个连接会先有数据到来。这样的话阻塞IO调用就不见得合适了。

> 一个反面例子：

```
/* This won't work. */
char buf[1024];
int i, n;
while (i_still_want_to_read()) {
    for (i=0; i<n_sockets; ++i) {
        n = recv(fd[i], buf, sizeof(buf), 0);
        if (n==0)
            handle_close(fd[i]);
        else if (n<0)
            handle_error(fd[i], errno);
        else
            handle_input(fd[i], buf, n);
    }
}
```

如果是fd[2]中先有数据到来，你的程序并不会尝试先去fd[2]中读取数据，而等待程序从fd[0]和fd[1]中读取到数据并完成读取后才会去读fd[2]中的数据。

有人会想用多线程或者多进程的方式去解决这个问题。一种简单的方式是使用单独的进程或者线程去处理每个独立连接，相当于每个连接都需要一个线程或者进程。这样的话单个连接的阻塞调用不会阻塞别的连接的线程或进程。

未完待续。。。

