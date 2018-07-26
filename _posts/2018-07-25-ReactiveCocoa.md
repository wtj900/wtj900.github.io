---
layout:     post
title:      ReactiveCocoa
subtitle:   RAC从入门到精通
date:       2018-07-25
author:     JT
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - RAC
---

转载：[一缕殇流化隐半边冰霜](http://www.jobbole.com/members/halfrost/)

# 前言

`ReactiveCocoa`是一个将函数响应式编程范例带入`Objective-C`的开源库。`ReactiveCocoa`是由`Josh Abernathy`和`Justin Spahr-Summers`两位大神在对`GitHub for Mac`的开发过程中编写的。`Justin Spahr-Summers`大神在2011年11月13号下午12点35分进行的第一次提交，直到2013年2月13日上午3点05分发布了其1.0 release，达到了第一个重要里程碑。`ReactiveCocoa`社区也非常活跃，目前最新版已经完成了`ReactiveCocoa 5.0.0-alpha.3`，目前在`5.0.0-alpha.4`开发中。

`ReactiveCocoa v2.5`是公认的`Objective-C`最稳定的版本，因此被广大的以OC为主要语言的客户端选中使用。`ReactiveCocoa v3.x`主要是基于`Swift 1.2`的版本，而`ReactiveCocoa v4.x`主要基于`Swift 2.x`，`ReactiveCocoa 5.0`就全面支持`Swift 3.0`，也许还有以后的`Swift 4.0`。接下来几篇博客先以`ReactiveCocoa v2.5`版本为例子，分析一下OC版的RAC具体实现（也许分析完了`RAC 5.0`就到来了）。也算是写在`ReactiveCocoa 5.0`正式版到来前夕的祝福吧。


# 什么是ReactiveCocoa？

`ReactiveCocoa`（其简称为RAC）是由Github 开源的一个应用于iOS和OS X开发的新框架。RAC具有函数式编程(FP)和响应式编程(RP)的特性。它主要吸取了`.Net`的`Reactive Extensions`的设计和实现。

`ReactiveCocoa`的宗旨是`Streams of values over time`，随着时间变化而不断流动的数据流。

`ReactiveCocoa`主要解决了以下这些问题：

* UI数据绑定

UI控件通常需要绑定一个事件，RAC可以很方便的绑定任何数据流到控件上。

* 用户交互事件绑定

RAC为可交互的UI控件提供了一系列能发送Signal信号的方法。这些数据流会在用户交互中相互传递。

* 解决状态以及状态之间依赖过多的问题

有了RAC的绑定之后，可以不用在关心各种复杂的状态，isSelect，isFinish……也解决了这些状态在后期很难维护的问题。

* 消息传递机制的大统一

OC中编程原来消息传递机制有以下几种：Delegate，Block Callback，Target-Action，Timers，KVO，objc上有一篇关于OC中这5种消息传递方式改如何选择的文章Communication Patterns，推荐大家阅读。现在有了RAC之后，以上这5种方式都可以统一用RAC来处理。


# RAC中的核心RACSignal

## 信号的订阅和发送

`ReactiveCocoa`中最核心的概念之一就是信号`RACStream`。`RACStream`中有两个子类——`RACSignal`和`RACSequence`。下面先来分析`RACSignal`。

我们会经常看到以下的代码：

```
RACSignal *signal = [RACSignal createSignal:
                     ^RACDisposable *(id subscriber)
{
    [subscriber sendNext:@1];
    [subscriber sendNext:@2];
    [subscriber sendNext:@3];
    [subscriber sendCompleted];
    return [RACDisposable disposableWithBlock:^{
        NSLog(@"signal dispose");
    }];
}];
RACDisposable *disposable = [signal subscribeNext:^(id x) {
    NSLog(@"subscribe value = %@", x);
} error:^(NSError *error) {
    NSLog(@"error: %@", error);
} completed:^{
    NSLog(@"completed");
}];
 
[disposable dispose];
```

这是一个RACSignal被订阅的完整过程。被订阅的过程中，究竟发生了什么？

```
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	return [RACDynamicSignal createSignal:didSubscribe];
}
```

RACSignal调用createSignal的时候，会调用RACDynamicSignal的createSignal的方法。

![](https://wtj900.github.io/img/RAC/RAC-Stream-tree.png)

RACDynamicSignal是RACSignal的子类。createSignal后面的参数是一个block。

`(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe`

block的返回值是RACDisposable类型，block名叫didSubscribe。block的唯一一个参数是id类型的subscriber，这个subscriber是必须遵循RACSubscriber协议的。

RACSubscriber是一个协议，其下有以下4个协议方法：

```
@protocol RACSubscriber <NSObject>
@required

- (void)sendNext:(id)value;
- (void)sendError:(NSError *)error;
- (void)sendCompleted;
- (void)didSubscribeWithDisposable:(RACCompoundDisposable *)disposable;

@end
```

所以新建Signal的任务就全部落在了RACSignal的子类RACDynamicSignal上了。

```
@interface RACDynamicSignal ()

// The block to invoke for each subscriber.
@property (nonatomic, copy, readonly) RACDisposable * (^didSubscribe)(id<RACSubscriber> subscriber);

@end
```

RACDynamicSignal这个类很简单，里面就保存了一个名字叫didSubscribe的block。

```
+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
	RACDynamicSignal *signal = [[self alloc] init];
	signal->_didSubscribe = [didSubscribe copy];
	return [signal setNameWithFormat:@"+createSignal:"];
}
```

这个方法中新建了一个RACDynamicSignal对象signal，并把传进来的didSubscribe这个block保存进刚刚新建对象signal里面的didSubscribe属性中。最后再给signal命名+createSignal:。

```
- (instancetype)setNameWithFormat:(NSString *)format, ... {
	if (getenv("RAC_DEBUG_SIGNAL_NAMES") == NULL) return self;

	NSCParameterAssert(format != nil);

	va_list args;
	va_start(args, format);

	NSString *str = [[NSString alloc] initWithFormat:format arguments:args];
	va_end(args);

	self.name = str;
	return self;
}
```

setNameWithFormat是RACStream里面的方法，由于RACDynamicSignal继承自RACSignal，所以它也能调用这个方法。

RACSignal的block就这样被保存起来了，那什么时候会被执行呢？

block闭包在订阅的时候才会被“释放”出来。

RACSignal调用subscribeNext方法，返回一个RACDisposable。

```
- (RACDisposable *)subscribeNext:(void (^)(id x))nextBlock error:(void (^)(NSError *error))errorBlock completed:(void (^)(void))completedBlock {
	NSCParameterAssert(nextBlock != NULL);
	NSCParameterAssert(errorBlock != NULL);
	NSCParameterAssert(completedBlock != NULL);
	
	RACSubscriber *o = [RACSubscriber subscriberWithNext:nextBlock error:errorBlock completed:completedBlock];
	return [self subscribe:o];
}
```

在这个方法中会新建一个RACSubscriber对象，并传入nextBlock，errorBlock，completedBlock。

```
@interface RACSubscriber ()

// These callbacks should only be accessed while synchronized on self.
@property (nonatomic, copy) void (^next)(id value);
@property (nonatomic, copy) void (^error)(NSError *error);
@property (nonatomic, copy) void (^completed)(void);

@property (nonatomic, strong, readonly) RACCompoundDisposable *disposable;

@end
```

RACSubscriber这个类很简单，里面只有4个属性，分别是nextBlock，errorBlock，completedBlock和一个RACCompoundDisposable信号。

```
+ (instancetype)subscriberWithNext:(void (^)(id x))next error:(void (^)(NSError *error))error completed:(void (^)(void))completed {
	RACSubscriber *subscriber = [[self alloc] init];

	subscriber->_next = [next copy];
	subscriber->_error = [error copy];
	subscriber->_completed = [completed copy];

	return subscriber;
}
```

subscriberWithNext方法把传入的3个block都保存分别保存到自己对应的block中。

RACSignal调用subscribeNext方法，最后return的时候，会调用[self subscribe:o]，这里实际是调用了RACDynamicSignal类里面的subscribe方法。

```
- (RACDisposable *)subscribe:(id<RACSubscriber>)subscriber {
	NSCParameterAssert(subscriber != nil);

	RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
	subscriber = [[RACPassthroughSubscriber alloc] initWithSubscriber:subscriber signal:self disposable:disposable];

	if (self.didSubscribe != NULL) {
		RACDisposable *schedulingDisposable = [RACScheduler.subscriptionScheduler schedule:^{
			RACDisposable *innerDisposable = self.didSubscribe(subscriber);
			[disposable addDisposable:innerDisposable];
		}];

		[disposable addDisposable:schedulingDisposable];
	}
	
	return disposable;
}
```

RACDisposable有3个子类，其中一个就是RACCompoundDisposable。

![](https://wtj900.github.io/img/RAC/RAC-Disposable-tree.png)

```
@interface RACCompoundDisposable : RACDisposable

+ (instancetype)compoundDisposable;

+ (instancetype)compoundDisposableWithDisposables:(NSArray *)disposables;

- (void)addDisposable:(RACDisposable *)disposable;

- (void)removeDisposable:(RACDisposable *)disposable;

@end
```

RACCompoundDisposable虽然是RACDisposable的子类，但是它里面可以加入多个RACDisposable对象，在必要的时候可以一口气都调用dispose方法来销毁信号。当RACCompoundDisposable对象被dispose的时候，也会自动dispose容器内的所有RACDisposable对象。

RACPassthroughSubscriber是一个私有的类。

RACPassthroughSubscriber类就只有这一个方法。目的就是为了把所有的信号事件从一个订阅者subscriber传递给另一个还没有disposed的订阅者subscriber。

RACPassthroughSubscriber类中保存了3个非常重要的对象，RACSubscriber，RACSignal，RACCompoundDisposable。RACSubscriber是待转发的信号的订阅者subscriber。RACCompoundDisposable是订阅者的销毁对象，一旦它被disposed了，innerSubscriber就再也接受不到事件流了。

这里需要注意的是内部还保存了一个RACSignal，并且它的属性是unsafe_unretained。这里和其他两个属性有区别， 其他两个属性都是strong的。这里之所以不是weak，是因为引用RACSignal仅仅只是一个DTrace probes动态跟踪技术的探针。如果设置成weak，会造成没必要的性能损失。所以这里仅仅是unsafe_unretained就够了。

```
- (instancetype)initWithSubscriber:(id<RACSubscriber>)subscriber signal:(RACSignal *)signal disposable:(RACCompoundDisposable *)disposable {
	NSCParameterAssert(subscriber != nil);

	self = [super init];
	if (self == nil) return nil;

	_innerSubscriber = subscriber;
	_signal = signal;
	_disposable = disposable;

	[self.innerSubscriber didSubscribeWithDisposable:self.disposable];
	return self;
}
```

回到RACDynamicSignal类里面的subscribe方法中，现在新建好了RACCompoundDisposable和RACPassthroughSubscriber对象了。

RACScheduler.subscriptionScheduler是一个全局的单例。

```
+ (instancetype)subscriptionScheduler {
	static dispatch_once_t onceToken;
	static RACScheduler *subscriptionScheduler;
	dispatch_once(&onceToken, ^{
		subscriptionScheduler = [[RACSubscriptionScheduler alloc] init];
	});

	return subscriptionScheduler;
}
```

RACScheduler再继续调用schedule方法。

```
- (RACDisposable *)schedule:(void (^)(void))block {
	NSCParameterAssert(block != NULL);

	if (RACScheduler.currentScheduler == nil) return [self.backgroundScheduler schedule:block];

	block();
	return nil;
}
```

```
+ (BOOL)isOnMainThread {
	return [NSOperationQueue.currentQueue isEqual:NSOperationQueue.mainQueue] || [NSThread isMainThread];
}

+ (instancetype)currentScheduler {
	RACScheduler *scheduler = NSThread.currentThread.threadDictionary[RACSchedulerCurrentSchedulerKey];
	if (scheduler != nil) return scheduler;
	if ([self.class isOnMainThread]) return RACScheduler.mainThreadScheduler;

	return nil;
}
```

在取currentScheduler的过程中，会判断currentScheduler是否存在，和是否在主线程中。如果都没有，那么就会调用后台backgroundScheduler去执行schedule。

schedule的入参就是一个block，执行schedule的时候会去执行block。也就是会去执行：

```
RACDisposable *innerDisposable = self.didSubscribe(subscriber);
[disposable addDisposable:innerDisposable];
```

这两句关键的语句。之前信号里面保存的block就会在此处被“释放”执行。self.didSubscribe(subscriber)这一句就执行了信号保存的didSubscribe闭包。

在didSubscribe闭包中有sendNext，sendError，sendCompleted，执行这些语句会分别调用RACPassthroughSubscriber里面对应的方法。

```
- (void)sendNext:(id)value {
	if (self.disposable.disposed) return;

	if (RACSIGNAL_NEXT_ENABLED()) {
		RACSIGNAL_NEXT(cleanedSignalDescription(self.signal), cleanedDTraceString(self.innerSubscriber.description), cleanedDTraceString([value description]));
	}

	[self.innerSubscriber sendNext:value];
}

- (void)sendError:(NSError *)error {
	if (self.disposable.disposed) return;

	if (RACSIGNAL_ERROR_ENABLED()) {
		RACSIGNAL_ERROR(cleanedSignalDescription(self.signal), cleanedDTraceString(self.innerSubscriber.description), cleanedDTraceString(error.description));
	}

	[self.innerSubscriber sendError:error];
}

- (void)sendCompleted {
	if (self.disposable.disposed) return;

	if (RACSIGNAL_COMPLETED_ENABLED()) {
		RACSIGNAL_COMPLETED(cleanedSignalDescription(self.signal), cleanedDTraceString(self.innerSubscriber.description));
	}

	[self.innerSubscriber sendCompleted];
}
```

这个时候的订阅者是RACPassthroughSubscriber。RACPassthroughSubscriber里面的innerSubscriber才是最终的实际订阅者，RACPassthroughSubscriber会把值再继续传递给innerSubscriber。

```
- (void)sendNext:(id)value {
	@synchronized (self) {
		void (^nextBlock)(id) = [self.next copy];
		if (nextBlock == nil) return;

		nextBlock(value);
	}
}

- (void)sendError:(NSError *)e {
	@synchronized (self) {
		void (^errorBlock)(NSError *) = [self.error copy];
		[self.disposable dispose];

		if (errorBlock == nil) return;
		errorBlock(e);
	}
}

- (void)sendCompleted {
	@synchronized (self) {
		void (^completedBlock)(void) = [self.completed copy];
		[self.disposable dispose];

		if (completedBlock == nil) return;
		completedBlock();
	}
}
```

innerSubscriber是RACSubscriber，调用sendNext的时候会先把自己的self.next闭包copy一份，再调用，而且整个过程还是线程安全的，用@synchronized保护着。最终订阅者的闭包在这里被调用。

sendError和sendCompleted也都是同理。

![](https://wtj900.github.io/img/RAC/RAC-subscribe-process.png)

1. RACSignal调用subscribeNext方法，新建一个RACSubscriber。
2. 新建的RACSubscriber会copy，nextBlock，errorBlock，completedBlock存在自己的属性变量中。
3. RACSignal的子类RACDynamicSignal调用subscribe方法。
4. 新建RACCompoundDisposable和RACPassthroughSubscriber对象。RACPassthroughSubscriber分别保存对RACSignal，RACSubscriber，RACCompoundDisposable的引用，注意对RACSignal的引用是unsafe_unretained的。
5. RACDynamicSignal调用didSubscribe闭包。先调用RACPassthroughSubscriber的相应的sendNext，sendError，sendCompleted方法。
6. RACPassthroughSubscriber再去调用self.innerSubscriber，即RACSubscriber的nextBlock，errorBlock，completedBlock。注意这里调用同样是先copy一份，再调用闭包执行。


## 基础操作
### 1. bind（变换）

概要：变换操作，可以对原始信号的值进行变换。

在RACSignal的源码里面包含了两个基本操作，concat和zipWith。不过在分析这两个操作之前，先来分析一下更加核心的一个函数，bind操作。

先来说说bind函数的作用：

1. 会订阅原始的信号。
2. 任何时刻原始信号发送一个值，都会在绑定的block转换一次。
3. 一旦绑定的block转换了值变成信号，就立即订阅，并把值发给订阅者subscriber。
4. 一旦绑定的block要终止绑定，原始的信号就complete。
5. 当所有的信号都complete，发送completed信号给订阅者subscriber。
6. 如果中途信号出现了任何error，都要把这个错误发送给subscriber

```
- (RACSignal *)bind:(RACStreamBindBlock (^)(void))block {
	NSCParameterAssert(block != NULL);

	/*
	 * -bind: should:
	 * 
	 * 1. Subscribe to the original signal of values.
	 * 2. Any time the original signal sends a value, transform it using the binding block.
	 * 3. If the binding block returns a signal, subscribe to it, and pass all of its values through to the subscriber as they're received.
	 * 4. If the binding block asks the bind to terminate, complete the _original_ signal.
	 * 5. When _all_ signals complete, send completed to the subscriber.
	 * 
	 * If any signal sends an error at any point, send that to the subscriber.
	 */

	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACStreamBindBlock bindingBlock = block();

		NSMutableArray *signals = [NSMutableArray arrayWithObject:self];

		RACCompoundDisposable *compoundDisposable = [RACCompoundDisposable compoundDisposable];

		void (^completeSignal)(RACSignal *, RACDisposable *) = ^(RACSignal *signal, RACDisposable *finishedDisposable) {
			BOOL removeDisposable = NO;

			@synchronized (signals) {
				[signals removeObject:signal];

				if (signals.count == 0) {
					[subscriber sendCompleted];
					[compoundDisposable dispose];
				} else {
					removeDisposable = YES;
				}
			}

			if (removeDisposable) [compoundDisposable removeDisposable:finishedDisposable];
		};

		void (^addSignal)(RACSignal *) = ^(RACSignal *signal) {
			@synchronized (signals) {
				[signals addObject:signal];
			}

			RACSerialDisposable *selfDisposable = [[RACSerialDisposable alloc] init];
			[compoundDisposable addDisposable:selfDisposable];

			RACDisposable *disposable = [signal subscribeNext:^(id x) {
				[subscriber sendNext:x];
			} error:^(NSError *error) {
				[compoundDisposable dispose];
				[subscriber sendError:error];
			} completed:^{
				@autoreleasepool {
					completeSignal(signal, selfDisposable);
				}
			}];

			selfDisposable.disposable = disposable;
		};

		@autoreleasepool {
			RACSerialDisposable *selfDisposable = [[RACSerialDisposable alloc] init];
			[compoundDisposable addDisposable:selfDisposable];

			RACDisposable *bindingDisposable = [self subscribeNext:^(id x) {
				// Manually check disposal to handle synchronous errors.
				if (compoundDisposable.disposed) return;

				BOOL stop = NO;
				id signal = bindingBlock(x, &stop);

				@autoreleasepool {
					if (signal != nil) addSignal(signal);
					if (signal == nil || stop) {
						[selfDisposable dispose];
						completeSignal(self, selfDisposable);
					}
				}
			} error:^(NSError *error) {
				[compoundDisposable dispose];
				[subscriber sendError:error];
			} completed:^{
				@autoreleasepool {
					completeSignal(self, selfDisposable);
				}
			}];

			selfDisposable.disposable = bindingDisposable;
		}

		return compoundDisposable;
	}] setNameWithFormat:@"[%@] -bind:", self.name];
}
```

为了弄清楚bind函数究竟做了什么，写出测试代码：

```
 RACSignal *signal = [RACSignal createSignal:
                         ^RACDisposable *(id subscriber)
    {
        [subscriber sendNext:@1];
        [subscriber sendNext:@2];
        [subscriber sendNext:@3];
        [subscriber sendCompleted];
        return [RACDisposable disposableWithBlock:^{
            NSLog(@"signal dispose");
        }];
    }];

    RACSignal *bindSignal = [signal bind:^RACStreamBindBlock{
        return ^RACSignal *(NSNumber *value, BOOL *stop){
            value = @(value.integerValue * 2);
            return [RACSignal return:value];
        };
    }];

    [bindSignal subscribeNext:^(id x) {
        NSLog(@"subscribe value = %@", x);
    }];
```

由于前面第一章节详细讲解了RACSignal的创建和订阅的全过程，这个也为了方法讲解，创建RACDynamicSignal，RACCompoundDisposable，RACPassthroughSubscriber这些都略过，这里着重分析一下bind的各个闭包传递创建和订阅的过程。

为了防止接下来的分析会让读者看晕，这里先把要用到的block进行编号。

```
    RACSignal *signal = [RACSignal createSignal:
                         ^RACDisposable *(id subscriber)
    {
        // block 1
    }

    RACSignal *bindSignal = [signal bind:^RACStreamBindBlock{
        // block 2
        return ^RACSignal *(NSNumber *value, BOOL *stop){
            // block 3
        };
    }];

    [bindSignal subscribeNext:^(id x) {
        // block 4
    }];

- (RACSignal *)bind:(RACStreamBindBlock (^)(void))block {
        // block 5
    return [[RACSignal createSignal:^(id subscriber) {
        // block 6
        RACStreamBindBlock bindingBlock = block();
        NSMutableArray *signals = [NSMutableArray arrayWithObject:self];

        void (^completeSignal)(RACSignal *, RACDisposable *) = ^(RACSignal *signal, RACDisposable *finishedDisposable) {
        // block 7
        };

        void (^addSignal)(RACSignal *) = ^(RACSignal *signal) {
        // block 8
            RACDisposable *disposable = [signal subscribeNext:^(id x) {
            // block 9
            }];
        };

        @autoreleasepool {
            RACDisposable *bindingDisposable = [self subscribeNext:^(id x) {
                // block 10
                id signal = bindingBlock(x, &stop);

                @autoreleasepool {
                    if (signal != nil) addSignal(signal);
                    if (signal == nil || stop) {
                        [selfDisposable dispose];
                        completeSignal(self, selfDisposable);
                    }
                }
            } error:^(NSError *error) {
                [compoundDisposable dispose];
                [subscriber sendError:error];
            } completed:^{
                @autoreleasepool {
                    completeSignal(self, selfDisposable);
                }
            }];
        }
        return compoundDisposable;
    }] ;
}
```

1. 先创建信号signal，didSubscribe把block1 copy保存起来。

2. 当信号调用bind进行绑定，会调用block5，didSubscribe把block6 copy保存起来。

3. 当订阅者开始订阅bindSignal的时候，流程如下：

  * bindSignal执行didSubscribe的block，即执行block6。
  
  * 在block6 的第一句代码，就是调用RACStreamBindBlock bindingBlock = block()，这里的block是外面传进来的block2，于是开始调用block2。执行完block2，会返回一个RACStreamBindBlock的对象。
  
  * 由于是signal调用的bind函数，所以bind函数里面的self就是signal，在bind内部订阅了signal的信号。subscribeNext所以会执行block1。
  
  * 执行block1，sendNext调用订阅者subscriber的nextBlock，于是开始执行block10。
  
  * block10中会先调用bindingBlock，这个是之前调用block2的返回值，这个RACStreamBindBlock对象里面保存的是block3。所以开始调用block3。

  * 在block3中入参是一个value，这个value是signal中sendNext中发出来的value的值，在block3中可以对value进行变换，变换完成后，返回一个新的信号signal’。
  
  * 如果返回的signal’为空，则会调用completeSignal，即调用block7。block7中会发送sendCompleted。如果返回的signal’不为空，则会调用addSignal，即调用block8。block8中会继续订阅signal’。执行block9。

  * block9 中会sendNext，这里的subscriber是block6的入参，于是对subscriber调用sendNext，会调用到bindSignal的订阅者的block4中。
  
  * block9 中执行完sendNext，还会调用sendCompleted。这里的是在执行block9里面的completed闭包。completeSignal(signal, selfDisposable);然后又会调用completeSignal，即block7。

  * 执行完block7，就完成了一次从signal 发送信号sendNext的全过程。

bind整个流程就完成了。

### 2. concat（组合）

概要：组合操作，先发送第一个信号，再发送第二个信号。

写出测试代码：

```
    RACSignal *signal = [RACSignal createSignal:
                         ^RACDisposable *(id subscriber)
    {
        [subscriber sendNext:@1];
        [subscriber sendCompleted];
        return [RACDisposable disposableWithBlock:^{
            NSLog(@"signal dispose");
        }];
    }];


    RACSignal *signals = [RACSignal createSignal:
                         ^RACDisposable *(id subscriber)
    {
        [subscriber sendNext:@2];
        [subscriber sendNext:@3];
        [subscriber sendNext:@6];
        [subscriber sendCompleted];
        return [RACDisposable disposableWithBlock:^{
            NSLog(@"signal dispose");
        }];
    }];

    RACSignal *concatSignal = [signal concat:signals];

    [concatSignal subscribeNext:^(id x) {
        NSLog(@"subscribe value = %@", x);
    }];
```

concat操作就是把两个信号合并起来。注意合并有先后顺序。

![](https://wtj900.github.io/img/RAC/RAC-stream-concat.png)

```
- (RACSignal *)concat:(RACSignal *)signal {
	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACSerialDisposable *serialDisposable = [[RACSerialDisposable alloc] init];

		RACDisposable *sourceDisposable = [self subscribeNext:^(id x) {
			[subscriber sendNext:x];
		} error:^(NSError *error) {
			[subscriber sendError:error];
		} completed:^{
			RACDisposable *concattedDisposable = [signal subscribe:subscriber];
			serialDisposable.disposable = concattedDisposable;
		}];

		serialDisposable.disposable = sourceDisposable;
		return serialDisposable;
	}] setNameWithFormat:@"[%@] -concat: %@", self.name, signal];
}
```

合并前，signal和signals分别都把各自的didSubscribe保存copy起来。

合并之后，合并之后新的信号的didSubscribe会把block保存copy起来。

当合并之后的信号被订阅的时候：

* 调用新的合并信号的didSubscribe。
* 由于是第一个信号调用的concat方法，所以block中的self是前一个信号signal。合并信号的didSubscribe会先订阅signal。
* 由于订阅了signal，于是开始执行signal的didSubscribe，sendNext，sendError。
* 当前一个信号signal发送sendCompleted之后，就会开始订阅后一个信号signals，调用signals的didSubscribe。
* 由于订阅了后一个信号，于是后一个信号signals开始发送sendNext，sendError，sendCompleted。

这样两个信号就前后有序的拼接到了一起。

这里有二点需要注意的是：

* 只有当第一个信号完成之后才能收到第二个信号的值，因为第二个信号是在第一个信号completed的闭包里面订阅的，所以第一个信号不结束，第二个信号也不会被订阅。
* 两个信号concat在一起之后，新的信号的结束信号在第二个信号结束的时候才结束。看上图描述，新的信号的发送长度等于前面两个信号长度之和，concat之后的新信号的结束信号也就是第二个信号的结束信号。

### 3. zipWith（拉链）

概要：拉链操作，两个信号从开始一一匹配，只发送匹配完成的数据。

写出测试代码：

```
RACSignal *signal = [RACSignal createSignal:
                         ^RACDisposable *(id subscriber)
                         {
                             [subscriber sendNext:@1];
                             [subscriber sendNext:@2];
                             [subscriber sendNext:@3];
                             [subscriber sendNext:@4];
                             [subscriber sendNext:@5];
                             [subscriber sendCompleted];
                             return [RACDisposable disposableWithBlock:^{
                                 NSLog(@"signal dispose----");
                             }];
                         }];
    
    
    RACSignal *signals = [RACSignal createSignal:
                          ^RACDisposable *(id subscriber)
                          {
                              [subscriber sendNext:@"A"];
                              [subscriber sendNext:@"B"];
                              [subscriber sendNext:@"C"];
                              [subscriber sendNext:@"D"];
                              [subscriber sendCompleted];
                              return [RACDisposable disposableWithBlock:^{
                                  NSLog(@"signal dispose++++");
                              }];
                          }];
    
    RACSignal *zipSignal = [signal zipWith:signals];
    
    [zipSignal subscribeNext:^(id x) {
        NSLog(@"subscribe value = %@", x);
    }];
```

![](https://wtj900.github.io/img/RAC/RAC-stream-zip.png)

```
- (RACSignal *)zipWith:(RACSignal *)signal {
    NSCParameterAssert(signal != nil);

    return [[RACSignal createSignal:^(id subscriber) {
        __block BOOL selfCompleted = NO;
        NSMutableArray *selfValues = [NSMutableArray array];

        __block BOOL otherCompleted = NO;
        NSMutableArray *otherValues = [NSMutableArray array];

        void (^sendCompletedIfNecessary)(void) = ^{
            @synchronized (selfValues) {
                BOOL selfEmpty = (selfCompleted && selfValues.count == 0);
                BOOL otherEmpty = (otherCompleted && otherValues.count == 0);

                // 如果任意一个信号完成并且数组里面空了，就整个信号算完成
                if (selfEmpty || otherEmpty) [subscriber sendCompleted];
            }
        };

        void (^sendNext)(void) = ^{
            @synchronized (selfValues) {

                // 数组里面的空了就返回。
                if (selfValues.count == 0) return;
                if (otherValues.count == 0) return;

                // 每次都取出两个数组里面的第0位的值，打包成元组
                RACTuple *tuple = RACTuplePack(selfValues[0], otherValues[0]);
                [selfValues removeObjectAtIndex:0];
                [otherValues removeObjectAtIndex:0];

                // 把元组发送出去
                [subscriber sendNext:tuple];
                sendCompletedIfNecessary();
            }
        };

        // 订阅第一个信号
        RACDisposable *selfDisposable = [self subscribeNext:^(id x) {
            @synchronized (selfValues) {

                // 把第一个信号的值加入到数组中
                [selfValues addObject:x ?: RACTupleNil.tupleNil];
                sendNext();
            }
        } error:^(NSError *error) {
            [subscriber sendError:error];
        } completed:^{
            @synchronized (selfValues) {

                // 订阅完成时判断是否要发送完成信号
                selfCompleted = YES;
                sendCompletedIfNecessary();
            }
        }];

        // 订阅第二个信号
        RACDisposable *otherDisposable = [signal subscribeNext:^(id x) {
            @synchronized (selfValues) {

                // 把第二个信号加入到数组中
                [otherValues addObject:x ?: RACTupleNil.tupleNil];
                sendNext();
            }
        } error:^(NSError *error) {
            [subscriber sendError:error];
        } completed:^{
            @synchronized (selfValues) {

                // 订阅完成时判断是否要发送完成信号
                otherCompleted = YES;
                sendCompletedIfNecessary();
            }
        }];

        return [RACDisposable disposableWithBlock:^{

            // 销毁两个信号
            [selfDisposable dispose];
            [otherDisposable dispose];
        }];
    }] setNameWithFormat:@"[%@] -zipWith: %@", self.name, signal];
}
```

当把两个信号通过zipWith之后，就像上面的那张图一样，拉链的两边被中间的拉索拉到了一起。既然是拉链，那么一一的位置是有对应的，上面的拉链第一个位置只能对着下面拉链第一个位置，这样拉链才能拉到一起去。

具体实现：

zipWith里面有两个数组，分别会存储两个信号的值。

* 一旦订阅了zipWith之后的信号，就开始执行didSubscribe闭包。
* 在闭包中会先订阅第一个信号。这里假设第一个信号比第二个信号先发出一个值。第一个信号发出来的每一个值都会被加入到第一个数组中保存起来，然后调用sendNext( )闭包。在sendNext( )闭包中，会先判断两个数组里面是否都为空，如果有一个数组里面是空的，就return。由于第二个信号还没有发送值，即第二个信号的数组里面是空的，所以这里第一个值发送不出来。于是第一个信号被订阅之后，发送的值存储到了第一个数组里面了，没有发出去。
* 第二个信号的值紧接着发出来了，第二个信号每发送一次值，也会存储到第二个数组中，但是这个时候再调用sendNext( )闭包的时候，不会再return了，因为两个数组里面都有值了，两个数组的第0号位置都有一个值了。有值以后就打包成元组RACTuple发送出去。并清空两个数组0号位置存储的值。
* 以后两个信号每次发送一个，就先存储在数组中，只要有“配对”的另一个信号，就一起打包成元组RACTuple发送出去。从图中也可以看出，zipWith之后的新信号，每个信号的发送时刻是等于两个信号最晚发出信号的时刻。
* 新信号的完成时间，是当两者任意一个信号完成并且数组里面为空，就算完成了。所以最后第一个信号发送的5的那个值就被丢弃了。

第一个信号依次发送的1，2，3，4的值和第二个信号依次发送的A，B，C，D的值，一一的合在了一起，就像拉链把他们拉在一起。由于5没法配对，所以拉链也拉不上了。

## 变换操作

### 1. Map

在父类RACStream中定义的,map操作一般是用来信号变换的。

```
RACSignal *signal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:@1];
        [subscriber sendNext:@2];
        [subscriber sendNext:@3];
        [subscriber sendCompleted];
        return nil;
    }];
    
    RACSignal *mapSignal = [signal map:^id(NSNumber *value) {
        return @(value.integerValue * 10);
    }];
    
    [mapSignal subscribeNext:^(NSNumber *x) {
        NSLog(@"%@",x);
    } completed:^{
        NSLog(@"completed");
    }];
```

![](https://wtj900.github.io/img/RAC/RAC-stream-map.png)

来看看底层是如何实现的。

```
- (instancetype)map:(id (^)(id value))block {
	NSCParameterAssert(block != nil);

	Class class = self.class;
	
	return [[self flattenMap:^(id value) {
		return [class return:block(value)];
	}] setNameWithFormat:@"[%@] -map:", self.name];
}
```

这里实现代码比较严谨，先判断self的类型。因为RACStream的子类中会有一些子类会重写这些方法，所以需要判断self的类型，在回调中可以回调到原类型的方法中去。

由于本篇文章中我们都分析RACSignal的操作，所以这里的self.class是RACDynamicSignal类型的。相应的在return返回值中也返回class，即RACDynamicSignal类型的信号。

从map实现代码上来看，map实现是用了flattenMap函数来实现的。把map的入参闭包，放到了flattenMap的返回值中。

在来看看flattenMap的实现：

```
- (instancetype)flattenMap:(RACStream * (^)(id value))block {
	Class class = self.class;

	return [[self bind:^{
		return ^(id value, BOOL *stop) {
			id stream = block(value) ?: [class empty];
			NSCAssert([stream isKindOfClass:RACStream.class], @"Value returned from -flattenMap: is not a stream: %@", stream);

			return stream;
		};
	}] setNameWithFormat:@"[%@] -flattenMap:", self.name];
}
```

flattenMap算是对bind函数的一种封装。bind函数的是一个返回值RACStreamBindBlock类型的闭包。而flattenMap函数的入参是一个value，返回值RACStream类型的闭包。

在flattenMap中，返回block(value)的信号，如果信号为nil，则返回[class empty]。

先来看看为空的情况。当block(value)为空，返回[RACEmptySignal empty]，empty就是创建了一个RACEmptySignal类型的信号：

```
+ (RACSignal *)empty {
#ifdef DEBUG
    // Create multiple instances of this class in DEBUG so users can set custom
    // names on each.
    return [[[self alloc] init] setNameWithFormat:@"+empty"];
#else
    static id singleton;
    static dispatch_once_t pred;
 
    dispatch_once(&pred, ^{
        singleton = [[self alloc] init];
    });
 
    return singleton;
#endif
}
```

RACEmptySignal类型的信号又是什么呢？

```
- (RACDisposable *)subscribe:(id)subscriber {
    NSCParameterAssert(subscriber != nil);
 
    return [RACScheduler.subscriptionScheduler schedule:^{
        [subscriber sendCompleted];
    }];
}
```

RACEmptySignal是RACSignal的子类，一旦订阅它，它就会同步的发送completed完成信号给订阅者。

所以flattenMap返回一个信号，如果信号不存在，就返回一个completed完成信号给订阅者。

再来看看flattenMap返回的信号是怎么变换的。

block(value)会把原信号发送过来的value传递给flattenMap的入参。flattenMap的入参是一个闭包，闭包的参数也是value的：

```
^(id value) { return [class return:block(value)]; }
```

这个闭包返回一个信号，信号类型和原信号的类型一样，即RACDynamicSignal类型的，值是block(value)。这里的闭包是外面map传进来的：

```
^id(NSNumber *value) { return @([value intValue] * 10); }
```

在这个闭包中把原信号的value值传进去进行变换，变换结束之后，包装成和原信号相同类型的信号，返回。返回的信号作为bind函数的闭包的返回值。这样订阅新的map之后的信号就会拿到变换之后的值。

### 2. MapReplace

一般用法如下：

`RACSignal *signalB = [signalA mapReplace:@"A"];`

![](https://wtj900.github.io/img/RAC/RAC-stream-mapreplace.png)

效果是不管A信号发送什么值，都替换成@“A”。

```
- (instancetype)mapReplace:(id)object {
    return [[self map:^(id _) {
        return object;
    }] setNameWithFormat:@"[%@] -mapReplace: %@", self.name, [object rac_description]];
}
```

看底层源码就知道，它并不去关心原信号发送的是什么值，原信号发送什么值，都返回入参object的值。

### 3. reduceEach

reduce是减少，聚合在一起的意思，reduceEach就是每个信号内部都聚合在一起。

```
RACSignal *signalA = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
        [subscriber sendNext:[RACTuple tupleWithObjects:@(1), @(2), nil]];
        [subscriber sendNext:[RACTuple tupleWithObjects:@(3), @(4), nil]];
        [subscriber sendNext:[RACTuple tupleWithObjects:@(5), @(6), nil]];
        [subscriber sendCompleted];
        return nil;
    }];
    
    RACSignal *singnalB = [signalA reduceEach:^id(NSNumber *num1, NSNumber *num2){
        return @(num1.integerValue + num2.integerValue);
    }];
    
    [singnalB subscribeNext:^(id x) {
        NSLog(@"%@",x);
    }];
```

![](https://wtj900.github.io/img/RAC/RAC-stream-reduceEach.png)

reduceEach后面必须传入一个元组RACTuple类型，否则会报错。

```
- (instancetype)reduceEach:(id (^)())reduceBlock {
	NSCParameterAssert(reduceBlock != nil);

	__weak RACStream *stream __attribute__((unused)) = self;
	return [[self map:^(RACTuple *t) {
		NSCAssert([t isKindOfClass:RACTuple.class], @"Value from stream %@ is not a tuple: %@", stream, t);
		return [RACBlockTrampoline invokeBlock:reduceBlock withArguments:t];
	}] setNameWithFormat:@"[%@] -reduceEach:", self.name];
}
```

这里有两个断言，一个是判断传入的reduceBlock闭包是否为空，另一个断言是判断闭包的入参是否是RACTuple类型的。

```
@interface RACBlockTrampoline : NSObject
@property (nonatomic, readonly, copy) id block;
+ (id)invokeBlock:(id)block withArguments:(RACTuple *)arguments;
@end
```

RACBlockTrampoline就是一个保存了一个block闭包的对象，它会根据传进来的参数，动态的构造一个NSInvocation，并执行。

reduceEach把入参reduceBlock作为RACBlockTrampoline的入参invokeBlock传进去，以及每个RACTuple也传到RACBlockTrampoline中。

```
- (id)invokeWithArguments:(RACTuple *)arguments {
    SEL selector = [self selectorForArgumentCount:arguments.count];
    NSInvocation *invocation = [NSInvocation invocationWithMethodSignature:[self methodSignatureForSelector:selector]];
    invocation.selector = selector;
    invocation.target = self;
 
    for (NSUInteger i = 0; i < arguments.count; i++) {
        id arg = arguments[i];
        NSInteger argIndex = (NSInteger)(i + 2);
        [invocation setArgument:&arg atIndex:argIndex];
    }
 
    [invocation invoke];
 
    __unsafe_unretained id returnVal;
    [invocation getReturnValue:&returnVal];
    return returnVal;
}
```

第一步就是先计算入参一个元组RACTuple里面元素的个数。

```
- (SEL)selectorForArgumentCount:(NSUInteger)count {
    NSCParameterAssert(count > 0);
 
    switch (count) {
        case 0: return NULL;
        case 1: return @selector(performWith:);
        case 2: return @selector(performWith::);
        case 3: return @selector(performWith:::);
        case 4: return @selector(performWith::::);
        case 5: return @selector(performWith:::::);
        case 6: return @selector(performWith::::::);
        case 7: return @selector(performWith:::::::);
        case 8: return @selector(performWith::::::::);
        case 9: return @selector(performWith:::::::::);
        case 10: return @selector(performWith::::::::::);
        case 11: return @selector(performWith:::::::::::);
        case 12: return @selector(performWith::::::::::::);
        case 13: return @selector(performWith:::::::::::::);
        case 14: return @selector(performWith::::::::::::::);
        case 15: return @selector(performWith:::::::::::::::);
    }
 
    NSCAssert(NO, @"The argument count is too damn high! Only blocks of up to 15 arguments are currently supported.");
    return NULL;
}
```

可以看到最多支持元组内元素的个数是15个。

这里我们假设以元组里面有2个元素为例。	

```
- (id)performWith:(id)obj1 :(id)obj2 {
    id (^block)(id, id) = self.block;
    return block(obj1, obj2);
}
```

对应的Type Encoding如下:

| ARGUMENT  | RETURN VALUE | 0 | 1 | 2 | 3 |
|:---------:|:------------:|:-:|:-:|:-:|:-:|
| methodSignature | @ |  @ | : | @ | @ |

argument0和argument1分别对应着隐藏参数self和_cmd，所以对应着的类型是@和：，从argument2开始，就是入参的Type Encoding了。

所以在构造invocation的参数的时候，argIndex是要偏移2个位置的。即从(i + 2)开始设置参数。

当动态构造了一个invocation方法之后，[invocation invoke]调用这个动态方法，也就是执行了外部的reduceBlock闭包，闭包里面是我们想要信号变换的规则。

闭包执行结束得到returnVal返回值。这个返回值就是整个RACBlockTrampoline的返回值了。这个返回值也作为了map闭包里面的返回值。

接下去的操作就完全转换成了map的操作了。上面已经分析过map操作了，这里就不赘述了。

### 4. reduceApply

举个例子：

```
RACSignal *signalA = [RACSignal createSignal:
                         ^RACDisposable *(id subscriber)
                         {
                             id block = ^id(NSNumber *first,NSNumber *second,NSNumber *third) {
                                 return @(first.integerValue + second.integerValue * third.integerValue);
                             };
 
                             [subscriber sendNext:RACTuplePack(block,<a href='http://www.jobbole.com/members/0929outlook'>@2</a> , @3 , @8)];
                             [subscriber sendNext:RACTuplePack((id)(^id(NSNumber *x){return @(x.intValue * 10);}),@9,@10,@30)];
 
                             [subscriber sendCompleted];
                             return [RACDisposable disposableWithBlock:^{
                                 NSLog(@"signal dispose");
                             }];
                         }];
 
    RACSignal *signalB = [signalA reduceApply];
```

使用reduceApply的条件也是需要信号里面的值是元组RACTuple。不过这里和reduceEach不同的是，原信号中每个元祖RACTuple的第0位必须要为一个闭包，后面n位为这个闭包的入参，第0位的闭包有几个参数，后面就需要跟几个参数。

如上述例子中，第一个元组第0位的闭包有3个参数，所以第一个元组后面还要跟3个参数。第二个元组的第0位的闭包只有一个参数，所以后面只需要跟一个参数。

当然后面可以跟更多的参数，如第二个元组，闭包后面跟了3个参数，但是只有第一个参数是有效值，后面那2个参数是无效不起作用的。唯一需要注意的就是后面跟的参数个数一定不能少于第0位闭包入参的个数，否则就会报错。

上面例子输出:

```
26  // 26 = 2 + 3 * 8；
90  // 90 = 9 * 10；
```

看看底层实现：

```
- (RACSignal *)reduceApply {
    return [[self map:^(RACTuple *tuple) {
        NSCAssert([tuple isKindOfClass:RACTuple.class], @"-reduceApply must only be used on a signal of RACTuples. Instead, received: %@", tuple);
        NSCAssert(tuple.count > 1, @"-reduceApply must only be used on a signal of RACTuples, with at least a block in tuple[0] and its first argument in tuple[1]");
 
        // We can't use -array, because we need to preserve RACTupleNil
        NSMutableArray *tupleArray = [NSMutableArray arrayWithCapacity:tuple.count];
        for (id val in tuple) {
            [tupleArray addObject:val];
        }
        RACTuple *arguments = [RACTuple tupleWithObjectsFromArray:[tupleArray subarrayWithRange:NSMakeRange(1, tupleArray.count - 1)]];
 
        return [RACBlockTrampoline invokeBlock:tuple[0] withArguments:arguments];
    }] setNameWithFormat:@"[%@] -reduceApply", self.name];
}
```

这里也有2个断言，第一个是确保传入的参数是RACTuple类型，第二个断言是确保元组RACTuple里面的元素各种至少是2个。因为下面取参数是直接从1号位开始取的。

reduceApply做的事情和reduceEach基本是等效的，只不过变换规则的block闭包一个是外部传进去的，一个是直接打包在每个信号元组RACTuple中第0位中。

### 5. materialize

这个方法会把信号包装成RACEvent类型。

```
- (RACSignal *)materialize {
    return [[RACSignal createSignal:^(id subscriber) {
        return [self subscribeNext:^(id x) {
            [subscriber sendNext:[RACEvent eventWithValue:x]];
        } error:^(NSError *error) {
            [subscriber sendNext:[RACEvent eventWithError:error]];
            [subscriber sendCompleted];
        } completed:^{
            [subscriber sendNext:RACEvent.completedEvent];
            [subscriber sendCompleted];
        }];
    }] setNameWithFormat:@"[%@] -materialize", self.name];
}
```

sendNext会包装成[RACEvent eventWithValue:x]，error会包装成[RACEvent eventWithError:error]，completed会被包装成RACEvent.completedEvent。注意，当原信号error和completed，新信号都会发送sendCompleted。

### 6. dematerialize

这个操作是materialize的逆向操作。它会把包装成RACEvent信号重新还原为正常的值信号。

```
- (RACSignal *)dematerialize {
    return [[self bind:^{
        return ^(RACEvent *event, BOOL *stop) {
            switch (event.eventType) {
                case RACEventTypeCompleted:
                    *stop = YES;
                    return [RACSignal empty];
 
                case RACEventTypeError:
                    *stop = YES;
                    return [RACSignal error:event.error];
 
                case RACEventTypeNext:
                    return [RACSignal return:event.value];
            }
        };
    }] setNameWithFormat:@"[%@] -dematerialize", self.name];
}
```

这里的实现也用到了bind函数，它会把原信号进行一个变换。新的信号会根据event.eventType进行转换。RACEventTypeCompleted被转换成[RACSignal empty]，RACEventTypeError被转换成[RACSignal error:event.error]，RACEventTypeNext被转换成[RACSignal return:event.value]。

### 7. not

```
- (RACSignal *)not {
	return [[self map:^(NSNumber *value) {
		NSCAssert([value isKindOfClass:NSNumber.class], @"-not must only be used on a signal of NSNumbers. Instead, got: %@", value);

		return @(!value.boolValue);
	}] setNameWithFormat:@"[%@] -not", self.name];
}
```

not操作需要传入的值都是NSNumber类型的。不是NSNumber类型会报错。not操作会把每个NSNumber按照BOOL的规则，取非，当成新信号的值。

### 8. and

```
- (RACSignal *)and {
    return [[self map:^(RACTuple *tuple) {
        NSCAssert([tuple isKindOfClass:RACTuple.class], @"-and must only be used on a signal of RACTuples of NSNumbers. Instead, received: %@", tuple);
        NSCAssert(tuple.count > 0, @"-and must only be used on a signal of RACTuples of NSNumbers, with at least 1 value in the tuple");
 
        return @([tuple.rac_sequence all:^(NSNumber *number) {
            NSCAssert([number isKindOfClass:NSNumber.class], @"-and must only be used on a signal of RACTuples of NSNumbers. Instead, tuple contains a non-NSNumber value: %@", tuple);
 
            return number.boolValue;
        }]);
    }] setNameWithFormat:@"[%@] -and", self.name];
}
```

and操作需要原信号的每个信号都是元组RACTuple类型的，因为只有这样，RACTuple类型里面的每个元素的值才能进行&运算。

and操作里面有3处断言。第一处，判断入参是不是元组RACTuple类型的。第二处，判断RACTuple类型里面至少包含一个NSNumber。第三处，判断RACTuple里面是否都是NSNumber类型，有一个不符合，都会报错。

```
- (RACSequence *)rac_sequence {
    return [RACTupleSequence sequenceWithTupleBackingArray:self.backingArray offset:0];
}
```

RACTuple类型先转换成RACTupleSequence。

```
+ (instancetype)sequenceWithTupleBackingArray:(NSArray *)backingArray offset:(NSUInteger)offset {
    NSCParameterAssert(offset _tupleBackingArray = backingArray;
    seq->_offset = offset;
    return seq;
}
```

RACTuple类型先转换成RACTupleSequence，即存成了一个数组。

```
- (BOOL)all:(BOOL (^)(id))block {
    NSCParameterAssert(block != NULL);
 
    NSNumber *result = [self foldLeftWithStart:@YES reduce:^(NSNumber *accumulator, id value) {
        return @(accumulator.boolValue && block(value));
    }];
 
    return result.boolValue;
}
 
- (id)foldLeftWithStart:(id)start reduce:(id (^)(id, id))reduce {
    NSCParameterAssert(reduce != NULL);
 
    if (self.head == nil) return start;
 
    for (id value in self) {
        start = reduce(start, value);
    }
 
    return start;
}
```

for会遍历RACSequence里面存的每一个值，分别都去调用reduce( )闭包。start的初始值为YES。reduce( )闭包是：

`^(NSNumber *accumulator, id value) { return @(accumulator.boolValue && block(value)); }`

这里又会去调用block( )闭包：

`^(NSNumber *number) { return number.boolValue; }`

number是原信号RACTuple的第一个值。第一次循环reduce( )闭包是拿YES和原信号RACTuple的第一个值进行&计算。第二个循环reduce( )闭包是拿原信号RACTuple的第一个值和第二个值进行&计算，得到的值参与下一次循环，与第三个值进行&计算，如此下去。这也是折叠函数的意思，foldLeft从左边开始折叠。fold函数会从左至右，把RACTuple转换成的数组里面每个值都一个接着一个进行&计算。

每个RACTuple都map成这样的一个BOOL值。接下去信号就map成了一个新的信号。

### 9. or

```
- (RACSignal *)or {
    return [[self map:^(RACTuple *tuple) {
        NSCAssert([tuple isKindOfClass:RACTuple.class], @"-or must only be used on a signal of RACTuples of NSNumbers. Instead, received: %@", tuple);
        NSCAssert(tuple.count > 0, @"-or must only be used on a signal of RACTuples of NSNumbers, with at least 1 value in the tuple");
 
        return @([tuple.rac_sequence any:^(NSNumber *number) {
            NSCAssert([number isKindOfClass:NSNumber.class], @"-or must only be used on a signal of RACTuples of NSNumbers. Instead, tuple contains a non-NSNumber value: %@", tuple);
 
            return number.boolValue;
        }]);
    }] setNameWithFormat:@"[%@] -or", self.name];
}
```

or操作的实现和and操作的实现大体类似。3处断言的作用和and操作完全一致，这里就不再赘述了。or操作的重点在any函数的实现上。or操作的入参也必须是RACTuple类型的。

```
- (BOOL)any:(BOOL (^)(id))block {
    NSCParameterAssert(block != NULL);
 
    return [self objectPassingTest:block] != nil;
}
 
 
- (id)objectPassingTest:(BOOL (^)(id))block {
    NSCParameterAssert(block != NULL);
 
    return [self filter:block].head;
}
 
 
- (instancetype)filter:(BOOL (^)(id value))block {
    NSCParameterAssert(block != nil);
 
    Class class = self.class;
 
    return [[self flattenMap:^ id (id value) {
        if (block(value)) {
            return [class return:value];
        } else {
            return class.empty;
        }
    }] setNameWithFormat:@"[%@] -filter:", self.name];
}
```

any会依次判断RACTupleSequence数组里面的值，依次每个进行filter。如果value对应的BOOL值是YES，就转换成一个RACTupleSequence信号。如果对应的是NO，则转换成一个empty信号。

只要RACTuple为NO，就一直返回empty信号，直到BOOL值为YES，就返回1。map变换信号后变成成1。找到了YES之后的值就不会再判断了。如果没有找到YES，中间都是NO的话，一直遍历到数组最后一个，信号只能返回0。

### 10. any:

```
- (RACSignal *)any:(BOOL (^)(id object))predicateBlock {
    NSCParameterAssert(predicateBlock != NULL);
 
    return [[[self materialize] bind:^{
        return ^(RACEvent *event, BOOL *stop) {
            if (event.finished) {
                *stop = YES;
                return [RACSignal return:@NO];
            }
 
            if (predicateBlock(event.value)) {
                *stop = YES;
                return [RACSignal return:@YES];
            }
 
            return [RACSignal empty];
        };
    }] setNameWithFormat:@"[%@] -any:", self.name];
}
```

原信号会先经过materialize转换包装成RACEvent事件。依次判断predicateBlock(event.value)值的BOOL值，如果返回YES，就包装成RACSignal的新信号，发送YES出去，并且stop接下来的信号。如果返回NO，就返回[RACSignal empty]空信号。直到event.finished，返回[RACSignal return:@NO]。

所以any:操作的目的是找到第一个满足predicateBlock条件的值。找到了就返回YES的RACSignal的信号，如果没有找到，返回NO的RACSignal。

### 11. any

```
- (RACSignal *)any {
    return [[self any:^(id x) {
        return YES;
    }] setNameWithFormat:@"[%@] -any", self.name];
}
```

ny操作是any:操作中的一种情况。即predicateBlock闭包永远都返回YES，所以any操作之后永远都只能得到一个只发送一个YES的新信号。

### 12. all:

```
- (RACSignal *)all:(BOOL (^)(id object))predicateBlock {
    NSCParameterAssert(predicateBlock != NULL);
 
    return [[[self materialize] bind:^{
        return ^(RACEvent *event, BOOL *stop) {
            if (event.eventType == RACEventTypeCompleted) {
                *stop = YES;
                return [RACSignal return:@YES];
            }
 
            if (event.eventType == RACEventTypeError || !predicateBlock(event.value)) {
                *stop = YES;
                return [RACSignal return:@NO];
            }
 
            return [RACSignal empty];
        };
    }] setNameWithFormat:@"[%@] -all:", self.name];
}
```

all:操作和any:有点类似。原信号会先经过materialize转换包装成RACEvent事件。对原信号发送的每个信号都依次判断predicateBlock(event.value)是否是NO 或者event.eventType == RACEventTypeError。如果predicateBlock(event.value)返回NO或者出现了错误，新的信号都返回NO。如果一直都没出现问题，在RACEventTypeCompleted的时候发送YES。

all:可以用来判断整个原信号发送过程中是否有错误事件RACEventTypeError，或者是否存在predicateBlock为NO的情况。可以把predicateBlock设置成一个正确条件。如果原信号出现错误事件，或者不满足设置的错误条件，都会发送新信号返回NO。如果全过程都没有出错，或者都满足predicateBlock设置的条件，则一直到RACEventTypeCompleted，发送YES的新信号。

### 13. repeat

```
- (RACSignal *)repeat {
	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		return subscribeForever(self,
			^(id x) {
				[subscriber sendNext:x];
			},
			^(NSError *error, RACDisposable *disposable) {
				[disposable dispose];
				[subscriber sendError:error];
			},
			^(RACDisposable *disposable) {
				// Resubscribe.
			});
	}] setNameWithFormat:@"[%@] -repeat", self.name];
}
```

repeat操作返回一个subscribeForever闭包，闭包里面要传入4个参数。

```
static RACDisposable *subscribeForever (RACSignal *signal, void (^next)(id), void (^error)(NSError *, RACDisposable *), void (^completed)(RACDisposable *)) {
	next = [next copy];
	error = [error copy];
	completed = [completed copy];

	RACCompoundDisposable *compoundDisposable = [RACCompoundDisposable compoundDisposable];

	RACSchedulerRecursiveBlock recursiveBlock = ^(void (^recurse)(void)) {
		RACCompoundDisposable *selfDisposable = [RACCompoundDisposable compoundDisposable];
		[compoundDisposable addDisposable:selfDisposable];

		__weak RACDisposable *weakSelfDisposable = selfDisposable;

		RACDisposable *subscriptionDisposable = [signal subscribeNext:next error:^(NSError *e) {
			@autoreleasepool {
				error(e, compoundDisposable);
				[compoundDisposable removeDisposable:weakSelfDisposable];
			}

			recurse();
		} completed:^{
			@autoreleasepool {
				completed(compoundDisposable);
				[compoundDisposable removeDisposable:weakSelfDisposable];
			}

			recurse();
		}];

		[selfDisposable addDisposable:subscriptionDisposable];
	};

	// Subscribe once immediately, and then use recursive scheduling for any
	// further resubscriptions.
	recursiveBlock(^{
		RACScheduler *recursiveScheduler = RACScheduler.currentScheduler ?: [RACScheduler scheduler];

		RACDisposable *schedulingDisposable = [recursiveScheduler scheduleRecursiveBlock:recursiveBlock];
		[compoundDisposable addDisposable:schedulingDisposable];
	});

	return compoundDisposable;
}
```

subscribeForever有4个参数，第一个参数是原信号，第二个是传入的next闭包，第三个是error闭包，最后一个是completed闭包。

subscribeForever一进入这个函数就会调用recursiveBlock( )闭包，闭包中有一个recurse( )的入参的参数。在recursiveBlock( )闭包中对原信号RACSignal进行订阅。next，error，completed分别会先调用传进来的闭包。然后error，completed执行完error( )和completed( )闭包之后，还会继续再执行recurse( )，recurse( )是recursiveBlock的入参。

```
- (RACDisposable *)scheduleRecursiveBlock:(RACSchedulerRecursiveBlock)recursiveBlock {
    RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
 
    [self scheduleRecursiveBlock:[recursiveBlock copy] addingToDisposable:disposable];
    return disposable;
}
 
- (void)scheduleRecursiveBlock:(RACSchedulerRecursiveBlock)recursiveBlock addingToDisposable:(RACCompoundDisposable *)disposable {
    @autoreleasepool {
        RACCompoundDisposable *selfDisposable = [RACCompoundDisposable compoundDisposable];
        [disposable addDisposable:selfDisposable];
 
        __weak RACDisposable *weakSelfDisposable = selfDisposable;
 
        RACDisposable *schedulingDisposable = [self schedule:^{ // 此处省略 }];
 
        [selfDisposable addDisposable:schedulingDisposable];
    }
}
```

先取到当前的currentScheduler，即recursiveScheduler，执行scheduleRecursiveBlock，在这个函数中，会调用schedule函数。这里的recursiveScheduler是RACQueueScheduler类型的。

```
- (RACDisposable *)schedule:(void (^)(void))block {
    NSCParameterAssert(block != NULL);
 
    RACDisposable *disposable = [[RACDisposable alloc] init];
 
    dispatch_async(self.queue, ^{
        if (disposable.disposed) return;
        [self performAsCurrentScheduler:block];
    });
 
    return disposable;
}
```	

如果原信号没有disposed，dispatch_async会继续执行block，在这个block中还会继续原信号的发送。所以原信号只要没有error信号，disposable.disposed就不会返回YES，就会一直调用block。所以在subscribeForever的error和completed的最后都会调用recurse( )闭包。error调用recurse( )闭包是为了结束调用block，结束所有的信号。completed调用recurse( )闭包是为了继续调用block( )闭包，也就是repeat的本质。原信号会继续发送信号，如此无限循环下去。

### 14. retry:

```
- (RACSignal *)retry:(NSInteger)retryCount {
	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		__block NSInteger currentRetryCount = 0;
		return subscribeForever(self,
			^(id x) {
				[subscriber sendNext:x];
			},
			^(NSError *error, RACDisposable *disposable) {
				if (retryCount == 0 || currentRetryCount < retryCount) {
					// Resubscribe.
					currentRetryCount++;
					return;
				}

				[disposable dispose];
				[subscriber sendError:error];
			},
			^(RACDisposable *disposable) {
				[disposable dispose];
				[subscriber sendCompleted];
			});
	}] setNameWithFormat:@"[%@] -retry: %lu", self.name, (unsigned long)retryCount];
}
```

在retry:的实现中，和repeat实现的区别是中间加入了一个currentRetryCount值。如果currentRetryCount > retryCount的话，就会在error中调用[disposable dispose]，这样subscribeForever就不会再无限循环下去了。

所以retry:操作的用途就是在原信号在出现error的时候，重试retryCount的次数，如果依旧error，那么就会停止重试。

如果原信号没有发生错误，那么原信号在发送结束，subscribeForever也就结束了。retry:操作对于没有任何error的信号相当于什么都没有发生。

### 15. retry

```
- (RACSignal *)retry {
    return [[self retry:0] setNameWithFormat:@"[%@] -retry", self.name];
}
```

这里的retry操作就是一个无限重试的操作。因为retryCount设置成0之后，在error的闭包中中，retryCount 永远等于 0，原信号永远都不会被dispose，所以subscribeForever会一直无限重试下去。

同样的，如果对一个没有error的信号调用retry操作，也是不起任何作用的。

### 16. scanWithStart: reduceWithIndex: 

先写出测试代码:

```
RACSignal *signalA = [RACSignal createSignal:^RACDisposable *(id subscriber)
                         {
                             [subscriber sendNext:@1];
                             [subscriber sendNext:@1];
                             [subscriber sendNext:@4];
                             [subscriber sendNext:@6];
                             return [RACDisposable disposableWithBlock:^{
                             }];
                         }];
 
    RACSignal *signalB = [signalA scanWithStart:@(2) reduceWithIndex:^id(NSNumber * running, NSNumber * next, NSUInteger index) {
        return @(running.intValue * next.intValue + index);
    }];
```

结论：running最新的值，next当前信号值，index当前信号位置。

```
2    // 2 * 1 + 0 = 2
3    // 2 * 1 + 1 = 3
14   // 3 * 4 + 2 = 14
87   // 14 * 6 + 3 = 87
```

![](https://wtj900.github.io/img/RAC/RAC-stream-scanWithStart_reduceWithIndex.png)

```
- (instancetype)scanWithStart:(id)startingValue reduceWithIndex:(id (^)(id, id, NSUInteger))reduceBlock {
    NSCParameterAssert(reduceBlock != nil);
 
    Class class = self.class;
 
    return [[self bind:^{
        __block id running = startingValue;
        __block NSUInteger index = 0;
 
        return ^(id value, BOOL *stop) {
            running = reduceBlock(running, value, index++);
            return [class return:running];
        };
    }] setNameWithFormat:@"[%@] -scanWithStart: %@ reduceWithIndex:", self.name, [startingValue rac_description]];
}
```

scanWithStart这个变换由初始值，变换函数reduceBlock( )，和index步进的变量组成。原信号的每个信号都会由变换函数reduceBlock( )进行变换。index每次都是自增。变换的初始值是由入参startingValue传入的。

### 17. scanWithStart: reduce:

```
- (instancetype)scanWithStart:(id)startingValue reduce:(id (^)(id running, id next))reduceBlock {
    NSCParameterAssert(reduceBlock != nil);
 
    return [[self
             scanWithStart:startingValue
             reduceWithIndex:^(id running, id next, NSUInteger index) {
                 return reduceBlock(running, next);
             }]
            setNameWithFormat:@"[%@] -scanWithStart: %@ reduce:", self.name, [startingValue rac_description]];
}
```

scanWithStart: reduce:就是scanWithStart: reduceWithIndex: 的缩略版。变换函数也是外面闭包reduceBlock( )传进来的。只不过变换过程中不会使用index自增的这个变量。

### 18. aggregateWithStart: reduceWithIndex:

```
- (RACSignal *)aggregateWithStart:(id)start reduceWithIndex:(id (^)(id, id, NSUInteger))reduceBlock {
    return [[[[self
               scanWithStart:start reduceWithIndex:reduceBlock]
              startWith:start]
             takeLast:1]
            setNameWithFormat:@"[%@] -aggregateWithStart: %@ reduceWithIndex:", self.name, [start rac_description]];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-aggregateWithStart_reduceWithIndex.png)

aggregate是合计的意思。所以最后变换出来的信号只有最后一个值。
aggregateWithStart: reduceWithIndex:操作调用了scanWithStart: reduceWithIndex:，原理和它完全一致。不同的是多了两步额外的操作，一个是startWith:，一个是takeLast:1。startWith:是在scanWithStart: reduceWithIndex:变换之后的信号之前加上start信号。takeLast:1是取最后一个信号。takeLast:和startWith:的详细分析文章下面会详述。

值得注意的一点是，原信号如果没有发送complete信号，那么该函数就不会输出新的信号值。因为在一直等待结束。

### 19. aggregateWithStart: reduce:

```
- (RACSignal *)aggregateWithStart:(id)start reduce:(id (^)(id running, id next))reduceBlock {
    return [[self
             aggregateWithStart:start
             reduceWithIndex:^(id running, id next, NSUInteger index) {
                 return reduceBlock(running, next);
             }]
            setNameWithFormat:@"[%@] -aggregateWithStart: %@ reduce:", self.name, [start rac_description]];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-aggregateWithStart_reduce.png)

aggregateWithStart: reduce:调用aggregateWithStart: reduceWithIndex:函数，只不过没有只用index值。同样，如果原信号没有发送complete信号，也不会输出任何信号。

### 20. aggregateWithStartFactory: reduce:

```
- (RACSignal *)aggregateWithStartFactory:(id (^)(void))startFactory reduce:(id (^)(id running, id next))reduceBlock {
    NSCParameterAssert(startFactory != NULL);
    NSCParameterAssert(reduceBlock != NULL);

    return [[RACSignal defer:^{
        return [self aggregateWithStart:startFactory() reduce:reduceBlock];
    }] setNameWithFormat:@"[%@] -aggregateWithStartFactory:reduce:", self.name];
}
```

aggregateWithStartFactory: reduce:内部实现就是调用aggregateWithStart: reduce:，只不过入参多了一个产生start的startFactory( )闭包罢了。

### 21. collect

```
- (RACSignal *)collect {
    return [[self aggregateWithStartFactory:^{
        return [[NSMutableArray alloc] init];
    } reduce:^(NSMutableArray *collectedValues, id x) {
        [collectedValues addObject:(x ?: NSNull.null)];
        return collectedValues;
    }] setNameWithFormat:@"[%@] -collect", self.name];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-collect.png)

collect函数会调用aggregateWithStartFactory: reduce:方法。把所有原信号的值收集起来，保存在NSMutableArray中。

## 时间操作

![](https://wtj900.github.io/img/RAC/RAC-stream-map.png)

## 过滤操作

## 组合操作

## 高阶信号操作

## 同步操作

## 副作用操作

## 多线程操作

## 其他操作














