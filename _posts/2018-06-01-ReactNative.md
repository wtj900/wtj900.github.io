---
layout:     post
title:      ReactNative
subtitle:   ReactNative学习
date:       2018-06-01
author:     JT
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - RN
---

# 快速搭建ReactNative跨平台开发iOS

ReactNative可以进行跨平台开发iOS和安卓，Mac搭建简单如下:

* 安装HomeBrew
* 安装Node.js
* 安装watchman 和 flow
* 安装React Native CLI

1. 安装HomeBrew

这是安装包管理器，安装后可以方便后续安装包的安装，[HomeBrew官网安装命令](https://brew.sh)

终端命令:

`/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`

2. 安装Node.js

直接使用HomeBrew安装,可尝试使用终端命:

`brew install node`

3. 安装watchman 和 flow

watchman是React修改source文件的一个工具
安装终端命令














## 1 安装HomeBrew

![](https://wtj900.github.io/img/reverse/分析方法.png)

1. 工具分析：
    
    对界面结构、文件操作、网络请求等进行分析，获取软件界面组成结构。
    
2. 静态分析： 

   例如通过点击按钮，获取程序功能的具体实现；对不同功能函数的分析，获取应用代码的框架构成。
   
   只能获取有哪些函数，函数中执行了那些代码，如果要获取代码执行流程或参数传递，就要用到动态分析。
         
3. 动态分析：
   
   对目标程序进行debug，打印参数，一步步调试，获取执行流程，然后可以获取功能的实现，并增强功能。

## 2 应用平台和层次

逆向工程在windows、安卓、苹果等平台都有应用。

逆向工程的应用层次：

* 硬件层、内核驱动层：可以获取底层的具体实现和原理。
* 设备框架层：可以获取framework内部的具体代码实现，或调用某个API，就可以获取内部的实现操作。
* 应用层：对应用进行分析：界面组成、数据拦截、功能等。

## 3 作用

![](https://wtj900.github.io/img/reverse/学习逆向的作用.png)

## 4 学习需要基础

![](https://wtj900.github.io/img/reverse/学习基础.png)

## 5 相关工具

逆向工程用的的一些工具：

1. 监控工具：

* reveal：获取界面结构、层次和构成。
* snoop-it: 可以得到类里边的所有方法，还可以得到，网络操作、文件操作、加密函数的操作等
* intospy:

2. 反汇编工具：

通过分析二进制文件，生成汇编代码，进而转换为高级语言。例如：hopper、IDA。

3. 调试跟踪工具：

通过设置断点，调试跟踪当前程序运行状态。例如：gdb、lldb。

## 6 逆向流程

![](https://wtj900.github.io/img/reverse/逆向流程.png)

# 系统安全机制

ios系统安全架构图：

![](https://wtj900.github.io/img/reverse/安全架构.png)

* 加密引擎对设备秘钥、组秘钥和apple 根证书进行加密
* secure enclave：用于加密和解密。
* 用户分区加密不能关闭
* AES加密引擎：加密的key是和硬件有关的

## 1 安全启动链

![](https://wtj900.github.io/img/reverse/安全启动链.png)

## 2 系统软件授权

![](https://wtj900.github.io/img/reverse/软件系统授权.png)

## 3 应用代码签名

![](https://wtj900.github.io/img/reverse/应用代码签名.png)

## 4 运行时进程安全性

### 4.1 Sandbox

![](https://wtj900.github.io/img/reverse/沙盒机制.png)

### 4.2 DEP

![](https://wtj900.github.io/img/reverse/EDP.png)

### 4.3 ASLR

![](https://wtj900.github.io/img/reverse/ASLR.png)

## 5 数据加密保护

![](https://wtj900.github.io/img/reverse/加密和保护.png)

## 6 存在的问题

![](https://wtj900.github.io/img/reverse/安全问题.png)


# 认识越狱

![](https://wtj900.github.io/img/reverse/什么是越狱.png)

![](https://wtj900.github.io/img/reverse/系统文件目录结构.png)

![](https://wtj900.github.io/img/reverse/越狱的利与弊.png)

# Hook

