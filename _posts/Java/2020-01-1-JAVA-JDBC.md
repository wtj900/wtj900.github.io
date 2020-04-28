---
layout:     post
title:      设计模式
subtitle:   设计模式
date:       2019-01-23
author:     JT
header-img: img/post-bg-github-cup.jpg
catalog:    true
tags:
    - 数据库
---

## 传感器集锦

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
  
  
  
  