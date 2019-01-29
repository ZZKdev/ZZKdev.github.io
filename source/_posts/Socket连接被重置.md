---
title: Socket连接被重置
date: 2018-05-27 15:04:43
tags: [网络编程, bug]
---
## 一个神奇的bug
用c语言搭建了一个类似http服务器的东西，在返回response后，游览器连接被重置。
## 原因分析
### 第一次尝试
试了一下不关闭socket连接，果然连接没有被重置，但是页面一直在加载中。。。。
### 百度之后
查询到原因可能是服务器关闭连接时不太优雅，导致数据包没发完就关闭连接了。。
### 解决方法
使用`int shutdown(int sockfd,int how)`函数确保输出缓冲区的数据全部发出。下面是函数使用方法：
int shutdown(SOCKET s, int howto); 
sock 为需要断开的套接字，howto 为断开方式。
howto有以下取值：
* SD_RECEIVE：关闭接收操作，也就是断开输入流。
* SD_SEND：关闭发送操作，也就是断开输出流。
* SD_BOTH：同时关闭接收和发送操作。

shutdown在操作成功时返回0，在出现错误时返回-1（并置相应errno）
如果关闭读，则接受缓冲区的未读出的所有数据都将丢失，以后不会再接受任何数据
如果关闭写，如果输出缓冲区内有数据，则所有的数据将发送出去后将发送一个FIN信号
而closesocket则是关闭该socket，马上发送FIN信号，所有的未完成发送或者接受的数据都将被丢失.

```
# 示例
# 先关闭写，再关闭套接字
shutdown(socket, SD_SEND);
closesocket(socket);
```
