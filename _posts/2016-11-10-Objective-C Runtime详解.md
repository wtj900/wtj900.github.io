---
layout:     post
title:      Objective-C Runtime 详解
subtitle:   Runtime 详解
date:       2017-02-04
author:     JT
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Runtime
    - iOS
--- 

## 相关知识

![](https://wtj900.github.io/img/runtime/Runtime.jpg)
![](https://wtj900.github.io/img/runtime/数据结构.jpg)
![](https://wtj900.github.io/img/runtime/消息发送及转发机制.jpg)
![](https://wtj900.github.io/img/runtime/Method_Swizzling.jpg)

参考链接

- 原文：[Objective-C Runtime运行时](http://blog.jobbole.com/79566/?utm_source=blog.jobbole.com&utm_medium=relatedPosts)
- Apple官方文档：[Objective-C Runtime Programming Guide](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048)
- Apple开源代码：[Objective-C Runtime源码](https://opensource.apple.com/source/objc4/)
- [Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)