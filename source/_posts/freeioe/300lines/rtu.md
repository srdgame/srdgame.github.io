---
title: 300 Lines 系列之RTU功能
date: 2019-07-13 15:56:04
categories:
	- freeioe
	- 300lines
tags:
	- freeioe
	- 300lines
---

# 300 Lines 系列 --- RUT功能开发

## 什么是300 Lines 系列

基于FreeIOE开发一系列完整应用，将代码锁定在300行以内。用以展示如何进行FreeIOE应用开发。

### 上代码

[RTU应用](https://github.com/freeioe/freeioe_example_apps/blob/master/rtu/app.lua)


## 功能需求

应用只支持一个串口数据透传功能，如透传多个串口，需要多次安装应用。

### 支持串口配置

* 支持串口参数配置，如波特率，数据位，停止位等等。
* 支持可视化配置（安利一下冬笋云的可视化配置面板功能，[文档](https://github.com/thingsroot/docs/blob/master/Development/thingscloud/visual_config_ui/README.md)

### 支持TCP客户端和服务端模式

二选一模式，并不支持两种并存。

#### 支持客户端模式

* 可配置服务器地址和端口
* 连接失败、被断开后要重连
* 统计本次连接的收发字节数

#### 支持服务器模式

* 可配置本地监听地址和端口（默认是0.0.0.0，即所有网卡)
* 有新客户端连接后，踢掉之前的连接，保证只有一个客户端能进行数据首发
* 统计当前活跃客户端的首发字节数

## 功能设计

### 串口读写

FreeIOE有以下几种层面的串口读写接口:

* SerialChannel
* app.port.serial
* SerialDriver
* rs232

#### SerialChannel

此接口仿造SocketChannel（[文档](https://github.com/cloudwu/skynet/wiki/SocketChannel) ),支持两种模式：
1. 问答模式。数据永远都是一问一答，比较适合用以Modbus Master应用开发。
2. 异步模式。要求每个报文有自己的唯一ID，比较罕见的模式。我也不知道什么协议能用（瀑布汗!!!)

结论：适合做协议通讯，但是不适合做数据透传

#### app.port.serial

此接口其实是SerialChannel的上层封装，并支持了读取超时的逻辑。 (文档无，怒!!)

结论：还是只适合类Modbus的协议通讯

#### SerialDriver

rs232模块的轻度封装，增加了数据就绪回调功能。(文档无，怒!! × 2)

结论: 找到适合的接口，文档去哪里了？ 等我后补吧.....

#### rs232

librs232的lua接口模块，只是映射了串口的数据读取，串口参数控制的系统接口。 文档？不好意思没有 :-(  请参考[示例](https://github.com/srdgame/librs232/blob/master/doc/example.lua)

结论: 最原生，确实需要更多的代码进行控制。

### 网络读写

同串口读写，FreeIOE也有一些不同层面的接口:

* SocketChannel
* app.port.socket
* Socket
* lsocket

#### SocketChannel

没啥好说的, Skynet的SocketChannel模块，具体说明请参考[文档](https://github.com/cloudwu/skynet/wiki/SocketChannel)

结论: 同SerialChannel

#### app.port.socket

没啥好说的同 app.port.serial

#### Socket

Skynet的socket封装模块，[文档在此](https://github.com/cloudwu/skynet/wiki/Socket)

结论: 原生就是好，还有文档的。 （鄙视自己,明天就补文档.... 我是寒号鸟)

#### lsocket

云大移植lsocket模块，功能更多，[文档](https://github.com/cloudwu/lsocket/tree/master/doc)

结论：大杀器，比Socket支持更多的功能（如本地socket,获取本地网卡信息） 

### 接口选择结论

好吧，上面看的好烦，来让我们直面残酷的选择....
1. 使用SerialDriver
2. 使用skynet.Socket

## 代码实现

好了，到这里就只有干货了。我们谈谈整体应用结构。

### 应用基础类

可以叫基础类吧，毕竟我是C艹程序员。上[文档](https://github.com/freeioe/freeioe_app_api_book/blob/master/app/base/init.md)

我们需要实现的接口只有:

1. initialize: 初始化
2. on_start: 打开串口、发起连接/监听端口
3. on_close: 关闭连接和串口
4. on_run: 统计数据透传字节数

initialize, on_close, on_run这些就不罗嗦了，看代码就好，很简单。重点讲on_start函数。

### on_start做了什么？

* 创建透传数据依赖的设备模型实例
* 根据配置打开串口
* 设定串口数据接收处理函数
* 开启TCP连接/监听

### 串口数据接收监听

``` lua
port:start(function(data, err)
...
end)
```

这里设定的数据接收回调有两个参数: data, err
data是接收到的数据流，为空时说明发生了串口错误
err是串口错误信息

遇到串口错误怎么处理？ 当然是不处理（老板扣工资ing...)
这里应该做的是发送设备事件，让用户知道这个错误，并重启应用自身（参考应用开发手册）。如此老板会给你加薪的!!!

这部分代码只需要实现：
	* 检测一下连接是否存在，存在就将数据流发往socket。
	* 调用dump_comm保证调试时能看到数据流

### 网络连接（客户端 ）

无论是Client还是Server模式，这里都做了一个重启链接的方法 ``` function app:connect_proc ```

主要逻辑是递增式发起连接/监听请求，直到成功为止。 发起连接的时间间隔是不停的×2，来增加间隔时间，当然最大时间间隔不超过64秒。

而客户端模式只需要调用socket.open函数，调用成功后就转入watch_client_socket函数，进行无限制的数据接手并写入串口句柄。

### 网络监听（服务器)

服务方式要先进行端口监听,即调用socket.listen函数。 listen成功后调用socket.start(fd, function(fd, addr) end) 对连接进来的客户端进行处理。

这里要特别注意进来的客户端fd, 要调用一次socket.start(fd)， 让本应用来处理这个客户端句柄的数据流（skynet的设计机制是可以将这个句柄传输给其他服务进程，从而让另外一个服务进程处理数据流）。

服务器模式的下的watch_server_socket比客户端复杂的地方在于需要同时检测监听用的句柄和当然客户端句柄。

## 完善

这个应用本身只是做了一个简单的透传应用，逻辑也只是无限制透明传输，甚至服务器模式处理新连接的方式也只有踢掉前一个连接。因此如果想将这个应用做的更好：

1. 支持更多配置参数（Socket心跳，限制客户端登入地址，限制重连次数）
2. 支持简单的包解析（包头、包尾）
3. 支持安全认证

