---
layout: post
title:  "Linux 软件移植到 Windows 平台"
subtitle: "链接 Windows DLL"
date:   2020-3-23 11:13:45 +0800
tags:
  - Windows
  - cygwin
categories: [Windows]
comment: true
---

项目背景：最近做了一个需要跨平台的项目，要求能够支持 Linux 系统和 Windows 系统并且调用不同操作系统上的硬件驱动，为了减少工作量，我在 Linux 系统上开发了原型，并使用 cygwin/msys2 将其移植到 Windows 系统上。

# 1、Cygwin 简介

通常来说，我们在 Linux 下编程遵循 posix 标准，并且调用的是 Linux 提供的系统调用，使用 libc 函数库；但是在 Windows 下编程的时候我们需要使用 Windows 提供的系统调用，并且使用 Windows 提供的函数库。比如，在使用 socket 库编程的时候，在 Linux 中，你只需要`#include <sys/socket.h>`就可以使用 socket 库了，但是在 Windows 下，你需要：

```c
#include <winsock2.h>

int main(){
    WORD sockVersion = MAKEWORD(2,2);  
    WSADATA wsaData;  
    if(WSAStartup(sockVersion, &wsaData)!=0)  
    {  
        return 0;  
    } 
    /*
    	.....
    	do something just like in Unix
    */
    WSACleanup();
}
```

相比于在 Linux 系统下，还需要额外进行加载套接字`WSAStartup()`关闭套接字`WASCleanup()`的步骤。

幸运的是，大部分函数定义在 Linux 和 Windows 下是相同的。

为了解决这些差异，你有两种做法：

1. 手动做兼容，通过预定义宏针对不同系统做不同的预处理；这种方式的优点是，软件任然是原生的，没有额外的转换消耗，缺点是要花费较多的开发和适配时间；
2. 做一层转换，将所有对差异的函数调用做一层转换，转而去调用 Windows 系统下的函数；这种方式的优点是省时省力，缺点就是要消耗额外的转换时间；

Cygwin 采用的就是方法 2，Cygwin 提供了一个模拟层：`cygwin1.dll`，通过这个模拟层来转换函数调用（对 linux 的函数调用会进入`cygwin1.dll`，由它来完成对 Windows 函数的调用）。

通过`cygwin1.dll`可以看出，这是一个动态链接库，因此，我们的 Linux 程序需要在 Cygwin 下重新编译，将该库的 symbol 引入程序，这样才能做转换。

## 扩展阅读

### MinGW

Minimalist GNU for Windows，是一个用于开发**原生 Windows 应用**的开发环境。它主要提供了针对 win32 应用的 GCC、GNU binutils 等工具，以及对等于 Windows SDK 的头文件（的子集）和用于 MinGW 版本 linker 的库函数（so、a 等，而不是 lib、dll)。

**注意：**MinGW 只是提供了一个编译 Windows 程序的 Linux 环境，这意味着，你写的应用需要是 Windows 原生应用，而不能通过 Linux 应用直接移植。

# MSYS2 简介

msys2 是 Windows 下的一个软件分发与构建平台。它提供了类 Unix 的环境和命令行接口，以及方便易用的 pacman 包管理工具。

msys2 在是基于 Cygwin（POSIX 兼容层）并且使用 MinGW-w64 来为原生 Windows 软件提供更好的互通性，github wiki 上原文如下：

>It is an independent rewrite of MSys, based on modern Cygwin (POSIX compatibility layer) and MinGW-w64 with the aim of better interoperability with native Windows software.

相比于 Cygwin 的`cygwin1.dll`msys2 有一个`msys-2.0.dll`。

## 子系统

msys2 包含三个子系统以及它们对应的包仓库：msys2、mingw32 和 mingw64。

mingw 子系统提供了原生 windows 应用环境，这些程序与其他 Windows 程序合作的很好，并且与其他子系统独立；

msys2 子系统提供了模拟的 POSIX 兼容层（绝大多数）环境，以及包管理系统，文件系统等等；

**注意**：msys2 中每个子系统都有自己的编译工具链，msys2-devel，mingw-w64-i686-toolchain，mingw-w64-x86_64-toolchain。

总而言之，msys2 更像一个集成了 Cygwin 和 MinGW 优点的环境，是一个更集中的平台。在使用的时候要注意选择对应的子系统。

# cygwin 下编译并链接 Windows DLL

在 cygwin 下需要重新编译应用程序，并且在链接的时候将 Windows DLL 链接进去，这一步和 Linux 下的链接过程是一样的，可以参考我的 Makefile:

```makefile
CC = gcc
CFLAGS = -Wall -pthread -std=gnu99
DEBUG = -g3
INCLUDE = -I./src/include

# Linux build
WINDOWS = -I./src/driver/XA429_Windows_share
SOURCES = $(wildcard src/*.c src/driver/XA429_Windows_share/*.c)
OBJS = $(patsubst src/%.c,src/%.o,$(SOURCES))

linux: arinc429_adapter arinc429_adapter.1 clean_build

arinc429_adapter: $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^ -L./ -lA429DLL 

%.o: %.c
	$(CC) $(CFLAGS) $(DEBUG) $(INCLUDE) $(WINDOWS) -c $< -o $@

arinc429_adapter.1: arinc429_adapter
	help2man -s 1 ./arinc429_adapter -o $@-t
	chmod a=r $@-t
	mv -f $@-t $@

# clear
clean: clean_build
	rm arinc429_adapter
	rm arinc429_adapter.1

clean_build:
	rm -f $(OBJS)
```

# msys2 下编译并链接 DLL

与 Cygwin 相似，但是在这里注意要选用 msys2-devel 子系统。

# 动态加载 DLL

**注意**，在这里需要动态加载 DLL，加载方式和 POSIX 加载方式相同。

```c
void (*funcp)(void);        /* Pointer to function with no arguments */
void* libHandle = dlopen("./A429DLL.dll", RTLD_LAZY);

/* do something */
funcp = (void (*)(void)) dlsym(libHandle, argv[2]);
err = dlerror();
if (err != NULL)
    fatal("dlsym: %s", err);
/* Try calling the address returned by dlsym() as a function
   that takes no arguments */
(*funcp)();

dlclose(libHandle);
```

目前尝试静态加载的方式似乎是行不通的.

使用`ProcessExplorer`查看 DLL 未能自动加载

![](\pictures\cygwin_DLL.PNG)