---
layout:     post
title:      AFNetworking学习之路(四)
subtitle:   监控网络状态
date:       2017-10-21
author:     JT
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - AFNetworking
---

`AFNetworkReachabilityManager` 是对 `SystemConfiguration` 模块的封装，苹果的文档中也有一个类似的项目 [Reachability](https://developer.apple.com/library/content/samplecode/Reachability/Introduction/Intro.html) 这里对网络状态的监控跟苹果官方的实现几乎是完全相同的。

# AFNetworkReachabilityManager 的使用和实现

AFNetworkReachabilityManager 的使用还是非常简单的，只需要三个步骤，就基本可以完成对网络状态的监控。

* 初始化 AFNetworkReachabilityManager
* 调用 startMonitoring 方法开始对网络状态进行监控
* 设置 networkReachabilityStatusBlock 在每次网络状态改变时, 调用这个 block

## 初始化 AFNetworkReachabilityManager

在初始化方法中，使用 `SCNetworkReachabilityCreateWithAddress` 或者 `SCNetworkReachabilityCreateWithName` 生成一个 `SCNetworkReachabilityRef` 的引用。

```
+ (instancetype)managerForDomain:(NSString *)domain {
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithName(kCFAllocatorDefault, [domain UTF8String]);

    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];

    return manager;
}

+ (instancetype)managerForAddress:(const void *)address {
    SCNetworkReachabilityRef reachability = SCNetworkReachabilityCreateWithAddress(kCFAllocatorDefault, (const struct sockaddr *)address);
    AFNetworkReachabilityManager *manager = [[self alloc] initWithReachability:reachability];

    return manager;
}
```

1. 这两个方法会通过一个域名或者一个 `sockaddr_in` 的指针生成一个 `SCNetworkReachabilityRef`
2. 调用` - [AFNetworkReachabilityManager initWithReachability:]` 将生成的 `SCNetworkReachabilityRef` 引用传给 `networkReachability`
3. 设置一个默认的 `networkReachabilityStatus`

```
- (instancetype)initWithReachability:(SCNetworkReachabilityRef)reachability {
    self = [super init];
    if (!self) {
        return nil;
    }

    _networkReachability = CFRetain(reachability);
    self.networkReachabilityStatus = AFNetworkReachabilityStatusUnknown;

    return self;
}
```
> 当调用 `CFBridgingRelease(reachability)` 后，会把 `reachability` 桥接成一个 `NSObject` 对象赋值给 `_networkReachability `，然后释放原来的`CoreFoundation` 对象。

## 监控网络状态

在初始化 AFNetworkReachabilityManager 后，会调用 startMonitoring 方法开始监控网络状态。

```
- (void)startMonitoring {
    [self stopMonitoring];

    if (!self.networkReachability) {
        return;
    }

    __weak __typeof(self)weakSelf = self;
    AFNetworkReachabilityStatusBlock callback = ^(AFNetworkReachabilityStatus status) {
        __strong __typeof(weakSelf)strongSelf = weakSelf;

        strongSelf.networkReachabilityStatus = status;
        if (strongSelf.networkReachabilityStatusBlock) {
            strongSelf.networkReachabilityStatusBlock(status);
        }

    };

    SCNetworkReachabilityContext context = {0, (__bridge void *)callback, AFNetworkReachabilityRetainCallback, AFNetworkReachabilityReleaseCallback, NULL};
    SCNetworkReachabilitySetCallback(self.networkReachability, AFNetworkReachabilityCallback, &context);
    SCNetworkReachabilityScheduleWithRunLoop(self.networkReachability, CFRunLoopGetMain(), kCFRunLoopCommonModes);

    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0),^{
        SCNetworkReachabilityFlags flags;
        if (SCNetworkReachabilityGetFlags(self.networkReachability, &flags)) {
            AFPostReachabilityStatusChange(flags, callback);
        }
    });
}
```

1、 先调用 `- stopMonitoring` 方法，如果之前设置过对网络状态的监听，使用 `SCNetworkReachabilityUnscheduleFromRunLoop` 方法取消之前在 `Main Runloop` 中的监听

```
- (void)stopMonitoring {
    if (!self.networkReachability) {
        return;
    }

    SCNetworkReachabilityUnscheduleFromRunLoop(self.networkReachability, CFRunLoopGetMain(), kCFRunLoopCommonModes);
}
```

2、 创建一个在每次网络状态改变时的回调,每次回调被调用时,重新设置 `networkReachabilityStatus` 属性,调用 `networkReachabilityStatusBlock`

```
__weak __typeof(self)weakSelf = self;
    AFNetworkReachabilityStatusBlock callback = ^(AFNetworkReachabilityStatus status) {
        __strong __typeof(weakSelf)strongSelf = weakSelf;

        strongSelf.networkReachabilityStatus = status;
        if (strongSelf.networkReachabilityStatusBlock) {
            strongSelf.networkReachabilityStatusBlock(status);
        }

    };
```

3、 创建一个 `SCNetworkReachabilityContext`

```
typedef struct {
     CFIndex		version;
     void *		__nullable info;
     const void	* __nonnull (* __nullable retain)(const void *info);
     void		(* __nullable release)(const void *info);
     CFStringRef	__nonnull (* __nullable copyDescription)(const void *info);
 } SCNetworkReachabilityContext;

 SCNetworkReachabilityContext context = {
     0,
     (__bridge void *)callback,
     AFNetworkReachabilityRetainCallback,
     AFNetworkReachabilityReleaseCallback,
     NULL
 };
```

* 其中的 `callback` 就是上一步中的创建的 `block` 对象
* 这里的 `AFNetworkReachabilityRetainCallback` 和 `AFNetworkReachabilityReleaseCallback` 都是非常简单的 block，在回调被调用时，只是使用 `Block_copy` 和 `Block_release` 这样的宏
* 传入的 `info` 会以参数的形式在 `AFNetworkReachabilityCallback` 执行时传入
`static const void * AFNetworkReachabilityRetainCallback(const void *info) { return Block_copy(info); }
static void AFNetworkReachabilityReleaseCallback(const void *info) { if (info) { Block_release(info); } }`

4、 当目标的网络状态改变时，会调用传入的回调

```
SCNetworkReachabilitySetCallback(
     (__bridge SCNetworkReachabilityRef)networkReachability,
     AFNetworkReachabilityCallback,
     &context
 );
```

5、 在 Main Runloop 中对应的模式开始监控网络状态

```
SCNetworkReachabilityScheduleWithRunLoop(
     (__bridge SCNetworkReachabilityRef)networkReachability,
     CFRunLoopGetMain(),
     kCFRunLoopCommonModes
 );
```

6、 获取当前的网络状态，调用 callback

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0),^{
     SCNetworkReachabilityFlags flags;
     if (SCNetworkReachabilityGetFlags((__bridge SCNetworkReachabilityRef)networkReachability, &flags)) {
         AFPostReachabilityStatusChange(flags, callback);
     }
 });
```

## 设置 networkReachabilityStatusBlock 以及回调

在 `Main Runloop` 中对网络状态进行监控之后，在每次网络状态改变，就会调用 `AFNetworkReachabilityCallback` 函数：

```
static void AFNetworkReachabilityCallback(SCNetworkReachabilityRef __unused target, SCNetworkReachabilityFlags flags, void *info) {
    AFPostReachabilityStatusChange(flags, (__bridge AFNetworkReachabilityStatusBlock)info);
}
```
这里会从 `info` 中取出之前存在 `context` 中的 `AFNetworkReachabilityStatusBlock`。

```
__weak __typeof(self)weakSelf = self;
AFNetworkReachabilityStatusBlock callback = ^(AFNetworkReachabilityStatus status) {
    __strong __typeof(weakSelf)strongSelf = weakSelf;

    strongSelf.networkReachabilityStatus = status;
    if (strongSelf.networkReachabilityStatusBlock) {
        strongSelf.networkReachabilityStatusBlock(status);
    }

};
```

取出这个 `block` 之后，传入 `AFPostReachabilityStatusChange` 函数：

```
static void AFPostReachabilityStatusChange(SCNetworkReachabilityFlags flags, AFNetworkReachabilityStatusBlock block) {
    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusForFlags(flags);
    dispatch_async(dispatch_get_main_queue(), ^{
        if (block) {
            block(status);
        }
        NSNotificationCenter *notificationCenter = [NSNotificationCenter defaultCenter];
        NSDictionary *userInfo = @{ AFNetworkingReachabilityNotificationStatusItem: @(status) };
        [notificationCenter postNotificationName:AFNetworkingReachabilityDidChangeNotification object:nil userInfo:userInfo];
    });
}
```

1. 调用 `AFNetworkReachabilityStatusForFlags` 获取当前的网络可达性状态
2. 在主线程中异步执行上面传入的 `callback block`（设置 self 的网络状态，调用 networkReachabilityStatusBlock）
3. 发送 AFNetworkingReachabilityDidChangeNotification 通知.

```
static AFNetworkReachabilityStatus AFNetworkReachabilityStatusForFlags(SCNetworkReachabilityFlags flags) {
    BOOL isReachable = ((flags & kSCNetworkReachabilityFlagsReachable) != 0);
    BOOL needsConnection = ((flags & kSCNetworkReachabilityFlagsConnectionRequired) != 0);
    BOOL canConnectionAutomatically = (((flags & kSCNetworkReachabilityFlagsConnectionOnDemand ) != 0) || ((flags & kSCNetworkReachabilityFlagsConnectionOnTraffic) != 0));
    BOOL canConnectWithoutUserInteraction = (canConnectionAutomatically && (flags & kSCNetworkReachabilityFlagsInterventionRequired) == 0);
    BOOL isNetworkReachable = (isReachable && (!needsConnection || canConnectWithoutUserInteraction));

    AFNetworkReachabilityStatus status = AFNetworkReachabilityStatusUnknown;
    if (isNetworkReachable == NO) {
        status = AFNetworkReachabilityStatusNotReachable;
    }
#if	TARGET_OS_IPHONE
    else if ((flags & kSCNetworkReachabilityFlagsIsWWAN) != 0) {
        status = AFNetworkReachabilityStatusReachableViaWWAN;
    }
#endif
    else {
        status = AFNetworkReachabilityStatusReachableViaWiFi;
    }

    return status;
}
```

因为 flags 是一个 SCNetworkReachabilityFlags，它的不同位代表了不同的网络可达性状态，通过 flags 的位操作，获取当前的状态信息 AFNetworkReachabilityStatus。

```
typedef CF_OPTIONS(uint32_t, SCNetworkReachabilityFlags) {
	kSCNetworkReachabilityFlagsTransientConnection	= 1<<0,
	kSCNetworkReachabilityFlagsReachable		= 1<<1,
	kSCNetworkReachabilityFlagsConnectionRequired	= 1<<2,
	kSCNetworkReachabilityFlagsConnectionOnTraffic	= 1<<3,
	kSCNetworkReachabilityFlagsInterventionRequired	= 1<<4,
	kSCNetworkReachabilityFlagsConnectionOnDemand	= 1<<5,	// __OSX_AVAILABLE_STARTING(__MAC_10_6,__IPHONE_3_0)
	kSCNetworkReachabilityFlagsIsLocalAddress	= 1<<16,
	kSCNetworkReachabilityFlagsIsDirect		= 1<<17,
#if	TARGET_OS_IPHONE
	kSCNetworkReachabilityFlagsIsWWAN		= 1<<18,
#endif	// TARGET_OS_IPHONE

	kSCNetworkReachabilityFlagsConnectionAutomatic	= kSCNetworkReachabilityFlagsConnectionOnTraffic
};
```

