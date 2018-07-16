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

安装终端命令 `brew install watchman`

flow是一个JavaScript 的静态类型检查器

安装终端命令 `brew install flow`

关于`brew update` && `brew upgrade`命令

官方文档建议定期运行`brew update` && `brew upgrade`来使您的应用程序保持最新状态。

4. 安装React Native CLI

这是个用来开发React Native的命令行工具

安装终端命令： `npm install -g react-native-cli`

哈哈~大功告成，来一个跨平台项目.

在当前目录新建一个项目

`react-native init HelloReactNative`

完成之后打开这一个项目,同时看到了android和ios