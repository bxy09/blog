---
title: nc (netcat) 的常见用法
date: 2016-03-11 16:55:54
tags:
---

nc 是一个很好用的网络工具，在所有系统中都能找到它。他的主要用途是通过 tcp 、与 udp 读写数据。
因此有人用它来进行文件传输、端口转发、端口扫描、网络聊天等工作。

<!--more-->

# 参数简单介绍
```
nc [options] [hostname] [port[s]]
```
在此只介绍之后会用到的一些参数，更多的参数请参考 man。
```
[-l] 作为服务器监听端口
[-p port] 指定本地端口
[-v] 展示连接详细信息
[-w timeout] 超时时间设置（秒），包括连接超时以及标准输入的超时
[-z] 扫描端口，不传输任何数据

```

# 常见应用
## 端口扫描

扫描192.168.228.222 1-1000端口
```
nc -vzw 1 192.168.228.222 1-1000
```
`-v` 展示详细信息，可以展示失败信息。

`-z` 关闭输入输出，方便扫描。

`-w 1` 加快扫描速度，跳过超时连接。

## 单页面服务

```
while true; do nc -l 8080 < a.html; done
```
在8080端口打开服务，展示单文件网页`a.html`。

## 网络聊天
```
#A:
nc -lp 1234
#B:
nc hostname 1234
```
A为服务器，通过`p`指定服务端口。
B为客户端，连接服务器的特定端口。

注意，在 mac 系统中，服务器命令为 `nc -l 1234`，不需要`-p`。

## 文件传输
```
#A:
nc -lp 1234 >  file
#B:
nc -w 1 hostname 1234 < file
```
文件从B发送到A，其中 `-w` 参数使得文件传输完毕之后连接关闭，
该参数很多时候不必要，因为文件末尾的EOF就可以关闭通道。

还是要注意mac下监听端口不需要`-p`参数。

## 目录传输
```
#A:
nc -l 1234 |tar xzvf -
#B:
tar czvf - dir|nc hostname 1234
```
文件夹从B发送到A，传输方式为目录打包压缩以后再传输，到达目的地后再解压。
## 端口转发
```
mknod backpipe p
nc -lp port1 0< backpipe | nc host2 port2 > backpipe
```

创建管道 `backpipe` 的作用是将转发后的结果通过标准输出灌入到管道，再送回到监听端返回回去。
可以通过 `tee` 复制 `backpipe` 将其也打印出来，方便调试，如：

```
mknod backpipe p
nc -lp port1 0< backpipe | nc host2 port2 |tee backpipe
```

以上是一个将监听到的信息通过一个客户端发送出去的转发器。这是我们最常见的需求。还可以任意组合得到其他组合。


```
#Server
mknod backpipe p
nc -lp port1 0< backpipe | nc -lp port2 > backpipe
#ClientA
nc host1 port1
#ClientB
nc host1 port2
```
Server实现本地端口到端口的转发。可以当做是另外两个客户端A、B之间通信的中介。
