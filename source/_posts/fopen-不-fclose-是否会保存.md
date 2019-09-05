---
title: fopen 不 fclose 是否会保存
date: 2019-09-05 14:57:16
tags: [c, linux]
---
## 前言

这个问题起源于我和同学的一次打赌，在我过去的认知中文件用 `fopen` 打开后就一定要用 `fclose` 关闭，否则将不能保存写入的内容，写入的数据会存留在缓冲区中。但是经过实际测验后，不用 `fclose` 写入的内容也能够保存... 痛失一瓶可乐...

在那时，我把它的原因归结于是操作系统自己去保存的没有再深究，今天看到一点别的东西，突然想起来可能那时的想法是错误的，这可能是要归功与编译器，与操作系统无关。

> ps : 在这里，我只讨论 linux 下 gcc 的情况

## main 和 _start

可能很多人都不知道，我们的程序执行的入口函数其实并不是 main 函数，而是从 _start 函数开始执行的。

来我们测验一下，先写一个简单的程序

```c
int main(void)
{
    return 0;
}
```

对的就这么简单就可以了，编译然后用 readelf 命令查看一下程序入口地址

```shell
gcc example.c -o test.o
readelf -h test.o
```

我们得到以下输出

```
ELF 头：
  Magic：  7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  类别:                              ELF64
  数据:                              2 补码，小端序 (little endian)
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI 版本:                          0
  类型:                              DYN (共享目标文件)
  系统架构:                          Advanced Micro Devices X86-64
  版本:                              0x1
  入口点地址：              0x1020
  程序头起点：              64 (bytes into file)
  Start of section headers:          14576 (bytes into file)
  标志：             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         11
  Size of section headers:           64 (bytes)
  Number of section headers:         27
  Section header string table index: 26
```

重点看程序入口地址那一行为 0x1020

我们将编译后的可执行文件用 objdump 反汇编看看，为了方便我将它输出重定向到文件里面来看

```shell
objdump -d test.o > test.s
```

可以看到 0x1020 这里刚好就是 .text 段的开始也是 _start 函数的入口地地址

```assembly
Disassembly of section .text:
0000000000001020 <_start>:                                                                                                       
1020:   f3 0f 1e fa                         endbr64
1024:   31 ed                                  xor    %ebp,%ebp
1026:   49 89 d1                            mov    %rdx,%r9
1029:   5e                                        pop    %rsi
102a:   48 89 e2                            mov    %rsp,%rdx
102d:   48 83 e4 f0                       and    $0xfffffffffffffff0,%rsp
1031:   50                                        push   %rax
1032:   54                                        push   %rsp
1033:   4c 8d 05 66 01 00 00    lea    0x166(%rip),%r8        # 11a0 <__libc_csu_fini>
103a:   48 8d 0d ef 00 00 00     lea    0xef(%rip),%rcx        # 1130 <__libc_csu_init>
1041:   48 8d 3d d1 00 00 00    lea    0xd1(%rip),%rdi        # 1119 <main>
1048:   ff 15 92 2f 00 00              callq  *0x2f92(%rip)        # 3fe0 <__libc_start_main@GLIBC_2.2.5>
104e:   f4                      hlt
104f:   90                      nop
```

现在看来，_start 是入口函数已经是毋庸置疑了，问题是我们的 main 函数去哪里了？

## __libc_start_main

在上面一段汇编代码中，我们可以明显地看到 _start 函数调用了一个 __libc_start_main 的函数并且将 main 函数的地址存到了 rdi 寄存器中，答案八九就是在这里了，但是这个函数是动态链接的，我反汇编后并没有得到它的代码...

```assembly
1041:   48 8d 3d d1 00 00 00    lea    0xd1(%rip),%rdi        # 1119 <main> 0x1041 + 0xd1 刚好是 main 函数地址
1048:   ff 15 92 2f 00 00              callq  *0x2f92(%rip)        # 3fe0 <__libc_start_main@GLIBC_2.2.5>
```

所以我又将它编译了一下，不过这次用静态链接，不然看不到 __libc_start_main 的代码。

```shell
gcc example.c -o test.o -static
```

然后再用 objdump 反编译一下，这次反编译出来足足有 12 万行的汇编...

```shell
objdump -d test.o > test.s
```

然后在反汇编文件里买年直接搜索 <__libc_start_main> 函数，可以找到下面几条关键代码

```assembly
##  具体流程是先将 rdi 寄存器中的 main 函数地址存放到 0x18(%rsp) 位置上，再将地址给寄存器 rax 用 callq 调用
401f6a:   48 89 7c 24 18          mov    %rdi,0x18(%rsp)                                                                   
4023c9:   48 8b 44 24 18          mov    0x18(%rsp),%rax                                                                    
4023ce:   ff d0                   callq  *%rax
# 之后调用了将 main 函数的返回值给 edi 寄存器，调用 exit 函数
4023d0:   89 c7                   mov    %eax,%edi                                                                          
4023d2:   e8 29 5f 00 00          callq  408300 <exit>
```

可以看出 main 函数是在 __libc_start_main 函数中调用的。

## exit 和 _exit

可以看出编译器在我们编译过程中链接了很多其他的东西，这个和我们之前的问题有什么关系呢？之前的分析可以得到我们的代码还链接了很多别的东西不仅有我们写的，从上面的汇编代码可以看出当我们调用完 main 函数后，__libc_start_main 函数会继续调用 exit 函数，而 exit 函数会**关闭所有打开的流，这将导致写所有被缓冲的输出**，删除用TMPFILE函数建立的所有临时文件。至此我们前面的问题就解决了，原因是调用了 exit 函数导致缓冲都输出到文件里面了。简而言之就是编译器给我们的主程序加了个 exit 函数。下面是大致过程

```assembly
_start:
	call __libc_start_main
_call__libc_start_main:
	call main
	call exit
```

说到 exit 就说一下 _exit 吧。

其实 exit 函数就是对 _exit 函数的一个封装，不过 exit 函数在调用 _exit 函数之前会调用终止程序（终止程序可以通过 atexit 函数注册），清除 IO 缓冲。

_exit 函数做了三件事：

- 关闭属于该进程的所有文件描述符
- 进程的任何子进程都由进程 init 继承
- 向进程的父节点发送 SIGCHLD 信号

如果我们将我们的程序这样写

```c
#include <stdio.h>
int main(void)
{
    FILE *fp = fopen("test", "w");
   	fwrite("123", 3, 1, fp);
   	_exit(0);
}
```

则文件内容不会保存。

## 纯净的程序?

gcc 提供了一系列的参数供我们使用我们也可以用 `nostartfiles` 指定不链接我们之前分析的启动例程

````shell
gcc test.c -e main -nostartfiles -o test.o
````

其中 -e 是用来指定程序入口的，由于我们现在不链接之前的启动例程所以编译器会找不到 _start 函数，我们必须自己指定一下入口。

现在可以用 objdump 反汇编看一下我们的程序，你可以看到尤为地简洁，十分纯净

```shell
objdump -d test.o
```

也可以将我们程序里面主函数名字随便换一下，换成 test_main，然后用 gcc 指定入口 test_main，这样我们就创建了一个”没有“主函数的程序但是可以运行的程序了。

```shell
gcc test.c -e test_main -nostartfiles
```

但是在程序运行结束时你应该会收到以下错误

```shell
[1]    10074 segmentation fault (core dumped)  ./a.out
```

出现这个错误是因为我们的程序不像之前我们有启动例程那样会调用 exit 正常退出，你可以自己在末尾加个 exit 或者 _exit 函数。

## 底层一点

`fopen` 函数底层调用`open`打开指定的文件，返回一个文件描述符（就是一个`int`类型的编号），分配一个`FILE`结构体，其中包含该文件的描述符、I/O缓冲区和当前读写位置等信息，返回这个`FILE`结构体的地址。就是因为 FILE 结构体这个缓冲区的存在我们才需要刷缓冲才能将文件写入，`fwrite`  `fread` 都是先看缓冲区是否满或空才决定使用 `write` 或 `read` 的。

之所以要使用缓冲区，是因为每次 `write` `read` 都是一次系统调用要进入内核，调用一个系统调用比用户调用要慢很多，在用户区开辟缓冲区可以有效减少系统调用，提升性能。

`open`、`read`、`write`、`close` 也称无缓冲 IO，如果我们之前的写入用 `write` 的话，即使不用 `close` 或 `exit` 它也会写进文件里面去，不用刷缓冲。有缓冲这么好，那我们什么时候要用无缓冲 IO 呢？

通常我们读写设备时通常是不希望有缓冲的，例如向代表网络设备的文件写数据就是希望数据通过网络设备发送出去，而不希望只写到缓冲区里就算完事儿了，当网络设备接收到数据时应用程序也希望第一时间被通知到，所以网络编程通常直接调用Unbuffered I/O函数。

> PS : 虽然 Unbuffered IO 函数在用户区没有缓冲区，但是内核中会有 IO 缓冲区。
