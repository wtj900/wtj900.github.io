---
layout:     post
title:      JAVA
subtitle:   JDBC
date:       2020-01-11
author:     JT
header-img: img/post-bg-github-cup.jpg
catalog:    true
tags:
    - JAVA
---

## JDBC

* 概念：Java DataBase Connectivity，Java数据库连接，Java语言操作数据库
* 本职：官方定义的一套操作所有关系型数据库的规则，即接口，各个数据库厂商实现接口，提供数据库驱动jar包。我们可以使用这套接口编程，真正执行的代码是驱动jar包中的实现类。
* 快速入门
	* 步骤：
		1. 导入驱动jar包，用什么数据库就导入什么jar包
			1. 复制jar包到项目的自建（如libs）目录下
			2. jar包-右键-add as library
		2. 注册驱动
		3. 获取数据库连接对象 connection
		4. 定义sql
		5. 执行sql语句，接受结果
		6. 处理结果
		7. 释放资源


[传感器集锦](https://www.jianshu.com/p/5fc26af852b6)
  
第一种方式：

```
[[NSNotificationCenter defaultCenter] addObserver:instance selector:@selector(brightnessDidChange:) name:UIScreenBrightnessDidChangeNotification object:nil];
```

第二种方式：

```
- (void)lightSensitive {
    
    // 1.获取硬件设备
    AVCaptureDevice *device = [AVCaptureDevice defaultDeviceWithDeviceType:AVCaptureDeviceTypeBuiltInWideAngleCamera mediaType:AVMediaTypeVideo position:AVCaptureDevicePositionFront];
//    AVCaptureDevice *device = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
//    device.position = AVCaptureDevicePositionFront;
    
    // 2.创建输入流
    AVCaptureDeviceInput *input = [[AVCaptureDeviceInput alloc] initWithDevice:device error:nil];
    
    // 3.创建设备输出流
    AVCaptureVideoDataOutput *output = [[AVCaptureVideoDataOutput alloc] init];
    [output setSampleBufferDelegate:self queue:dispatch_get_main_queue()];
    
    // AVCaptureSession属性
    self.session = [[AVCaptureSession alloc]init];
    // 设置为高质量采集率
    [self.session setSessionPreset:AVCaptureSessionPresetHigh];
    // 添加会话输入和输出
    if ([self.session canAddInput:input]) {
        [self.session addInput:input];
    }
    if ([self.session canAddOutput:output]) {
        [self.session addOutput:output];
    }
    
    // 9.启动会话
    [self.session startRunning];
    
}

#pragma mark- AVCaptureVideoDataOutputSampleBufferDelegate的方法
- (void)captureOutput:(AVCaptureOutput *)captureOutput didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection {
    
    CFDictionaryRef metadataDict = CMCopyDictionaryOfAttachments(NULL,sampleBuffer, kCMAttachmentMode_ShouldPropagate);
    NSDictionary *metadata = [[NSMutableDictionary alloc] initWithDictionary:(__bridge NSDictionary*)metadataDict];
    CFRelease(metadataDict);
    NSDictionary *exifMetadata = [[metadata objectForKey:(NSString *)kCGImagePropertyExifDictionary] mutableCopy];
    float brightnessValue = [[exifMetadata objectForKey:(NSString *)kCGImagePropertyExifBrightnessValue] floatValue];
    
    NSLog(@"%f",brightnessValue);
    [UIScreen mainScreen].brightness = brightnessValue;
    
    
//    // 根据brightnessValue的值来打开和关闭闪光灯
//    AVCaptureDevice *device = [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];
//    BOOL result = [device hasTorch];// 判断设备是否有闪光灯
//    if ((brightnessValue < 0) && result) {// 打开闪光灯
//
//        [device lockForConfiguration:nil];
//
//        [device setTorchMode: AVCaptureTorchModeOn];//开
//
//        [device unlockForConfiguration];
//
//    }else if((brightnessValue > 0) && result) {// 关闭闪光灯
//
//        [device lockForConfiguration:nil];
//        [device setTorchMode: AVCaptureTorchModeOff];//关
//        [device unlockForConfiguration];
//
//    }
    
}
```
  
  
  
  