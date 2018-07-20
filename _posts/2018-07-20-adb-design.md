---
title: ADB的架构设计
category: Android
---

`ADB`，即 [`Android Debug Bridge`](https://developer.android.com/studio/command-line/adb.html)，它是`Android`开发/测试人员不可替代的强大工具，也是 `Android`设备玩家的好玩具。如果想更多了解`ADB`，可以查看一个开源项目[`Awesome Adb`](https://github.com/mzlogin/awesome-adb)。本文不讲`ADB`的用法，只简单介绍下它的设计架构。

## 一.`ADB`代码地址
~~~bash
https://github.com/aosp-mirror/platform_system_core/tree/master/adb 
~~~

## 二.`adb`架构
![adb-design](/img/postimg/adb.png)

## 三.`adb`组件

#### 1.`adb server`
`adb server`进程运行在主机后台。它的目的是要感知USB端口以了解设备何时连接/移除，以及模拟器实例何时启动/停止。

#### 2.`adb daemon` (`adbd`)

'`adbd`'运行在`Android`设备/模拟器中中。其目的是连接到adb服务器（通过`USB`设备，通过`TCP`模拟器连接）并提供少许客户端运行在主机上的服务。
`adb`服务器连接`adbd`成功时则判断设备处于`online`状态，否则，该设备处于`offline`状态，这意味着`adb`服务器检测到新的设备/仿真器，但不能连接到`adbd`守护进程。

#### 3.`adb`命令行客户端

主要用于`adb`的`shell`环境或者脚本环境。它首先尝试在主机上找到adb服务器，如果没有找到，将自动启动一个。然后，客户端将其服务请求发送到`adb server`。

#### 4.服务

- `host service`

这种类型的服务在`adb server`中运行因此不需要与客户端通信，比较典型的是"`adb devices`"这个指令，它只需要返回当前已知的设备列表跟状态即可。

- `local services`
这些服务一般运行在`adbd daemon`中，或者运行在设备本身，他的角色包括创建`client`跟`server`端的连接，接着传输数据。

## 四.协议细节

#### 1.`Client` <-> `Server`协议

`adb server`监听`tcp:localhost:5037`端口。下面以从客户端发送一个查询内部版本的指令过程举例，整个流程如下：
- `adb client`连接到`tcp:localhost:5037`端口
- `client`发送字符串"`000Chost:version`"等待响应
- 大部分情况下任务结束后`socket`将保持`alive`状态，以备下次请求，除非指令会重定向到`daemon`进程中

#### 2.`Daemon` <-> `Server`（`Traceports`协议）

目前`traceports`支持两种形式
- `USB Transports`，用于物理层的`USB`连接
- `Local Traceports`，模拟器通过`TCP`协议与主机连接


