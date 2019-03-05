---
layout:     post
title:      AFNetworking学习之路(五)
subtitle:   AFURLSessionManager
date:       2017-10-21
author:     JT
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - AFNetworking
---

`AFURLSessionManager` 绝对可以称得上是 `AFNetworking` 的核心。

1. 负责创建和管理 `NSURLSession`
2. 管理 `NSURLSessionTask`
3. 实现 `NSURLSessionDelegate` 等协议中的代理方法
4. 使用 `AFURLSessionManagerTaskDelegate` 管理进度
5. 使用 `_AFURLSessionTaskSwizzling` 调剂方法
6. 引入 `AFSecurityPolicy` 保证请求的安全
7. 引入 `AFNetworkReachabilityManager` 监控网络状态


我们会在这里着重介绍上面七个功能中的前五个，分析它是如何包装 NSURLSession 以及众多代理方法的。

# 创建和管理 NSURLSession

在使用 AFURLSessionManager 时，第一件要做的事情一定是初始化：

```
#import "NSObject+BYModel.h"

- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    self = [super init];
    if (!self) {
        return nil;
    }

    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }

    self.sessionConfiguration = configuration;

    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;

    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];

    self.responseSerializer = [AFJSONResponseSerializer serializer];

    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

#if !TARGET_OS_WATCH
    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];
#endif

    self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];

    self.lock = [[NSLock alloc] init];
    self.lock.name = AFURLSessionManagerLockName;

    // 为已有的任务设置代理
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        for (NSURLSessionDataTask *task in dataTasks) {
            [self addDelegateForDataTask:task uploadProgress:nil downloadProgress:nil completionHandler:nil];
        }

        for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
            [self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
        }

        for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
            [self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
        }
    }];

    return self;
}
```

在初始化方法中，需要完成初始化一些自己持有的实例：

1. 初始化会话配置`（NSURLSessionConfiguration）`，默认为 `defaultSessionConfiguration`
2. 初始化会话（session），并设置会话的代理以及代理队列
3. 初始化管理响应序列化（AFJSONResponseSerializer），安全认证（AFSecurityPolicy）以及监控网络状态（AFNetworkReachabilityManager）的实例
4. 初始化保存 data task 的字典（mutableTaskDelegatesKeyedByTaskIdentifier）

# 管理 NSURLSessionTask

```
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler;

- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromFile:(NSURL *)fileURL
                                         progress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                                completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError  * _Nullable error))completionHandler;

...

- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                             progress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                                          destination:(nullable NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                    completionHandler:(nullable void (^)(NSURLResponse *response, NSURL * _Nullable filePath, NSError * _Nullable error))completionHandler;

...
```

这里省略了一些返回 `NSURLSessionTask` 的方法，因为这些接口的形式都是差不多的。

我们将以 `- [AFURLSessionManager dataTaskWithRequest:uploadProgress:downloadProgress:completionHandler:]` 方法的实现为例，分析它是如何实例化并返回一个 `NSURLSessionTask` 的实例的：

```
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler {

    __block NSURLSessionDataTask *dataTask = nil;
    url_session_manager_create_task_safely(^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

    [self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];

    return dataTask;
}
```

1. 调用 `- [NSURLSession dataTaskWithRequest:]` 方法传入 `NSURLRequest`
2. 调用 `- [AFURLSessionManager addDelegateForDataTask:uploadProgress:downloadProgress:completionHandler:]` 方法创建一个 `AFURLSessionManagerTaskDelegate` 对象
3. 将 `completionHandler` `uploadProgressBlock`和 `downloadProgressBlock` 传入该对象并在相应事件发生时进行回调

```
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    [self setDelegate:delegate forTask:dataTask];

    delegate.uploadProgressBlock = uploadProgressBlock;
    delegate.downloadProgressBlock = downloadProgressBlock;
}
```

在这个方法中同时调用了另一个方法 `- [AFURLSessionManager setDelegate:forTask:]` 来设置代理：

```
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{

	#1: 检查参数, 略

    [self.lock lock];
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    [delegate setupProgressForTask:task];
    [self addNotificationObserverForTask:task];
    [self.lock unlock];
}
```

正如上面所提到的，`AFURLSessionManager` 就是通过字典 `mutableTaskDelegatesKeyedByTaskIdentifier` 来存储并管理每一个 `NSURLSessionTask`，它以 `taskIdentifier` 为键存储 `task`。

该方法使用 `NSLock` 来保证不同线程使用 `mutableTaskDelegatesKeyedByTaskIdentifier` 时，不会出现线程竞争的问题。

同时调用 `- setupProgressForTask:`，我们会在下面具体介绍这个方法。

