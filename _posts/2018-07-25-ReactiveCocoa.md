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

![](https://wtj900.github.io/img/RAC/RAC-disposable-tree.png)

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

### 1. throttle:valuesPassingTest:

> throttle:节流，在确定的时间间隔中,发送一个信号,之后没有信号，时间结束时发出该信号；如果发送了一个新值，则旧值被丢弃，直到时间结束，发送最新的值；但是当闭包predicate不满足时，说明当前信号不屏蔽，直接输出。

这个操作传入一个时间间隔NSTimeInterval，和一个判断条件的闭包predicate。原信号在一个时间间隔NSTimeInterval之间发送的信号，如果还能满足predicate，则原信号都被“吞”了，直到一个时间间隔NSTimeInterval结束，会再次判断predicate，如果不满足了，原信号就会被发送出来。

![](https://wtj900.github.io/img/RAC/RAC-stream-throttle_valuesPassingTest.png)

如上图，原信号发送1以后，间隔NSTimeInterval的时间内，没有信号发出，并且predicate也为YES，就把1变换成新的信号发出去。接下去由于原信号发送2，3，4的过程中，都在间隔NSTimeInterval的时间内，所以都被“吞”了。直到原信号发送5之后，间隔NSTimeInterval的时间内没有新的信号发出，所以把原信号的5发送出来。原信号的6也是如此。

再来看看具体实现：

```
- (RACSignal *)throttle:(NSTimeInterval)interval valuesPassingTest:(BOOL (^)(id next))predicate {
    NSCParameterAssert(interval >= 0);
    NSCParameterAssert(predicate != nil);
 
    return [[RACSignal createSignal:^(id subscriber) {
        RACCompoundDisposable *compoundDisposable = [RACCompoundDisposable compoundDisposable];
 
        RACScheduler *scheduler = [RACScheduler scheduler];
 
        __block id nextValue = nil;
        __block BOOL hasNextValue = NO;
        RACSerialDisposable *nextDisposable = [[RACSerialDisposable alloc] init];
 
        void (^flushNext)(BOOL send) = ^(BOOL send) { // 暂时省略 };
 
        RACDisposable *subscriptionDisposable = [self subscribeNext:^(id x) {
            // 暂时省略
        } error:^(NSError *error) {
            [compoundDisposable dispose];
            [subscriber sendError:error];
        } completed:^{
            flushNext(YES);
            [subscriber sendCompleted];
        }];
 
        [compoundDisposable addDisposable:subscriptionDisposable];
        return compoundDisposable;
    }] setNameWithFormat:@"[%@] -throttle: %f valuesPassingTest:", self.name, (double)interval];
}
```

看这段实现，里面有2处断言。会先判断传入的interval是否大于0，小于0当然是不行的。还有一个就是传入的predicate闭包不能为空，这个是接下来用来控制流程的。

接下来的实现还是按照套路来，返回值是一个信号，新信号的闭包里面再订阅原信号进行变换。

那么整个变换的重点就落在了flushNext闭包和订阅原信号subscribeNext闭包中了。

当新的信号一旦被订阅，闭包执行到此处，就会对原信号进行订阅。

```
[self subscribeNext:^(id x) {
    RACScheduler *delayScheduler = RACScheduler.currentScheduler ?: scheduler;
    BOOL shouldThrottle = predicate(x);
 
    @synchronized (compoundDisposable) {   
        flushNext(NO);
        if (!shouldThrottle) {
            [subscriber sendNext:x];
            return;
        }
        nextValue = x;
        hasNextValue = YES;
        nextDisposable.disposable = [delayScheduler afterDelay:interval schedule:^{
            flushNext(YES);
        }];
    }
}
```

1. 首先先创建一个delayScheduler。先判断当前的currentScheduler是否存在，不存在就取之前创建的[RACScheduler scheduler]。这里虽然两处都是RACTargetQueueScheduler类型的，但是currentScheduler是com.ReactiveCocoa.RACScheduler.mainThreadScheduler，而[RACScheduler scheduler]创建的是com.ReactiveCocoa.RACScheduler.backgroundScheduler。
2. 调用predicate( )闭包，传入原信号发来的信号值x，经过predicate判断以后，得到是否打开节流开关的BOOL变量shouldThrottle。
3. 之所以把RACCompoundDisposable作为线程间互斥信号量，因为RACCompoundDisposable里面会加入所有的RACDisposable信号。接着下面的操作用@synchronized给线程间加锁。
4. flushNext( )这个闭包是为了hook住原信号的发送。

```
void (^flushNext)(BOOL send) = ^(BOOL send) {
    @synchronized (compoundDisposable) {
        [nextDisposable.disposable dispose];
 
        if (!hasNextValue) return;
        if (send) [subscriber sendNext:nextValue];
 
        nextValue = nil;
        hasNextValue = NO;
    }
};
```

这个闭包中如果传入的是NO，那么原信号就无法立即sendNext。如果传入的是YES，并且hasNextValue = YES，原信号待发送的还有值，那么就发送原信号。

shouldThrottle是一个阀门，随时控制原信号是否可以被发送。

小结一下，每个原信号发送过来，通过在throttle:valuesPassingTest:里面的did subscriber闭包中进行订阅。这个闭包中主要干了4件事情：

调用flushNext(NO)闭包判断能否发送原信号的值。入参为NO，不发送原信号的值。
判断阀门条件predicate(x)能否发送原信号的值。
如果以上两个条件都满足，nextValue中进行赋值为原信号发来的值，hasNextValue = YES代表当前有要发送的值。
开启一个delayScheduler，延迟interval的时间，发送原信号的这个值，即调用flushNext(YES)。

现在再来分析一下整个throttle:valuesPassingTest:的全过程

原信号发出第一个值，如果在interval的时间间隔内，没有新的信号发送，那么delayScheduler延迟interval的时间，执行flushNext(YES)，发送原信号的这个第一个值。

```
- (RACDisposable *)afterDelay:(NSTimeInterval)delay schedule:(void (^)(void))block {
    return [self after:[NSDate dateWithTimeIntervalSinceNow:delay] schedule:block];
}
 
- (RACDisposable *)after:(NSDate *)date schedule:(void (^)(void))block {
    NSCParameterAssert(date != nil);
    NSCParameterAssert(block != NULL);
 
    RACDisposable *disposable = [[RACDisposable alloc] init];
 
    dispatch_after([self.class wallTimeWithDate:date], self.queue, ^{
        if (disposable.disposed) return;
        [self performAsCurrentScheduler:block];
    });
 
    return disposable;
}
```

注意，在dispatch_after闭包里面之前[self performAsCurrentScheduler:block]之前，有一个关键的判断：

`if (disposable.disposed) return;`

这个判断就是用来判断从第一个信号发出，在间隔interval的时间之内，还有没有其他信号存在。如果有，第一个信号肯定会disposed，这里会执行return，所以也就不会把第一个信号发送出来了。

这样也就达到了节流的目的：原来每个信号都会创建一个delayScheduler，都会延迟interval的时间，在这个时间内，如果原信号再没有发送新值，即原信号没有disposed，就把原信号的值发出来；如果在这个时间内，原信号还发送了一个新值，那么第一个值就被丢弃。在发送过程中，每个信号都要判断一次predicate( )，这个是阀门的开关，如果随时都不节流了，原信号发的值就需要立即被发送出来。

还有二点需要注意的是，第一点，正好在interval那一时刻，有新信号发送出来，原信号也会被丢弃，即只有在>=interval的时间之内，原信号没有发送新值，原来的这个值才能发送出来。第二点，原信号发送completed时，会立即执行flushNext(YES)，把原信号的最后一个值发送出来。

### 2. throttle:

```
- (RACSignal *)throttle:(NSTimeInterval)interval {
    return [[self throttle:interval valuesPassingTest:^(id _) {
        return YES;
    }] setNameWithFormat:@"[%@] -throttle: %f", self.name, (double)interval];
}
```

这个操作其实就是调用了throttle:valuesPassingTest:方法，传入时间间隔interval，predicate( )闭包则永远返回YES，原信号的每个信号都执行节流操作。

### 3. bufferWithTime:onScheduler:

这个操作的实现是类似于throttle:valuesPassingTest:的实现。

```
- (RACSignal *)bufferWithTime:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler {
    NSCParameterAssert(scheduler != nil);
    NSCParameterAssert(scheduler != RACScheduler.immediateScheduler);
 
    return [[RACSignal createSignal:^(id subscriber) {
        RACSerialDisposable *timerDisposable = [[RACSerialDisposable alloc] init];
        NSMutableArray *values = [NSMutableArray array];
 
        void (^flushValues)() = ^{
            // 暂时省略
        };
 
        RACDisposable *selfDisposable = [self subscribeNext:^(id x) {
            // 暂时省略
        } error:^(NSError *error) {
            [subscriber sendError:error];
        } completed:^{
            flushValues();
            [subscriber sendCompleted];
        }];
 
        return [RACDisposable disposableWithBlock:^{
            [selfDisposable dispose];
            [timerDisposable dispose];
        }];
    }] setNameWithFormat:@"[%@] -bufferWithTime: %f onScheduler: %@", self.name, (double)interval, scheduler];
}
```

bufferWithTime:onScheduler:的实现和throttle:valuesPassingTest:的实现给出类似。开始有2个断言，2个都是判断scheduler的，第一个断言是判断scheduler是否为nil。第二个断言是判断scheduler的类型的，scheduler类型不能是immediateScheduler类型的，因为这个方法是要缓存一些信号的，所以不能是immediateScheduler类型的。

```
RACDisposable *selfDisposable = [self subscribeNext:^(id x) {
    @synchronized (values) {
        if (values.count == 0) {
            timerDisposable.disposable = [scheduler afterDelay:interval schedule:flushValues];
        }
        [values addObject:x ?: RACTupleNil.tupleNil];
    }
}
```

在subscribeNext中，当数组里面是没有存任何原信号的值，就会开启一个scheduler，延迟interval时间，执行flushValues闭包。如果里面有值了，就继续加到values的数组中。关键的也是闭包里面的内容，代码如下：

```
void (^flushValues)() = ^{
    @synchronized (values) {
        [timerDisposable.disposable dispose];
 
        if (values.count == 0) return;
 
        RACTuple *tuple = [RACTuple tupleWithObjectsFromArray:values];
        [values removeAllObjects];
        [subscriber sendNext:tuple];
    }
};
```

flushValues( )闭包里面主要是把数组包装成一个元组，并且全部发送出来，原数组里面就全部清空了。这也是bufferWithTime:onScheduler:的作用，在interval时间内，把这个时间间隔内的原信号都缓存起来，并且在interval的那一刻，把这些缓存的信号打包成一个元组，发送出来。

和throttle:valuesPassingTest:方法一样，在原信号completed的时候，立即执行flushValues( )闭包，把里面存的值都发送出来。

### 4. delay:

delay:函数的操作和上面几个套路都是一样的，实现方式也都是模板式的，唯一的不同都在subscribeNext中，和一个判断是否发送的闭包中。

```
- (RACSignal *)delay:(NSTimeInterval)interval {
    return [[RACSignal createSignal:^(id subscriber) {
        RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];

        // We may never use this scheduler, but we need to set it up ahead of
        // time so that our scheduled blocks are run serially if we do.
        RACScheduler *scheduler = [RACScheduler scheduler];

        void (^schedule)(dispatch_block_t) = ^(dispatch_block_t block) {
            // 暂时省略
        };

        RACDisposable *subscriptionDisposable = [self subscribeNext:^(id x) {
            // 暂时省略
        } error:^(NSError *error) {
            [subscriber sendError:error];
        } completed:^{
            schedule(^{
                [subscriber sendCompleted];
            });
        }];

        [disposable addDisposable:subscriptionDisposable];
        return disposable;
    }] setNameWithFormat:@"[%@] -delay: %f", self.name, (double)interval];
}
```

在delay:的subscribeNext中，就单纯的执行了schedule的闭包。

```
RACDisposable *subscriptionDisposable = [self subscribeNext:^(id x) {
            schedule(^{
                [subscriber sendNext:x];
            });
        }
```

```
void (^schedule)(dispatch_block_t) = ^(dispatch_block_t block) {
          RACScheduler *delayScheduler = RACScheduler.currentScheduler ?: scheduler;
          RACDisposable *schedulerDisposable = [delayScheduler afterDelay:interval schedule:block];
          [disposable addDisposable:schedulerDisposable];
      };
```

在schedule闭包中做的时间就是延迟interval的时间发送原信号的值。

### 5. interval:onScheduler:withLeeway:

```
+ (RACSignal *)interval:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler withLeeway:(NSTimeInterval)leeway {
    NSCParameterAssert(scheduler != nil);
    NSCParameterAssert(scheduler != RACScheduler.immediateScheduler);
 
    return [[RACSignal createSignal:^(id subscriber) {
        return [scheduler after:[NSDate dateWithTimeIntervalSinceNow:interval] repeatingEvery:interval withLeeway:leeway schedule:^{
            [subscriber sendNext:[NSDate date]];
        }];
    }] setNameWithFormat:@"+interval: %f onScheduler: %@ withLeeway: %f", (double)interval, scheduler, (double)leeway];
}
```

在这个操作中，实现代码不难。先来看看2个断言，都是保护入参类型的，scheduler不能为空，且不能是immediateScheduler的类型，原因和上面是一样的，这里是延迟操作。

主要的实现就在`after:repeatingEvery:withLeeway:schedule:`上了。

这里的实现就是用GCD在self.queue上创建了一个Timer，时间间隔是interval，修正时间是leeway。

leeway这个参数是为dispatch source指定一个期望的定时器事件精度，让系统能够灵活地管理并唤醒内核。例如系统可以使用leeway值来提前或延迟触发定时器，使其更好地与其它系统事件结合。创建自己的定时器时，应该尽量指定一个leeway值。不过就算指定leeway值为0，也不能完完全全期望定时器能够按照精确的纳秒来触发事件。

这个定时器在interval执行sendNext操作，也就是发送原信号的值。

### 6. interval:onScheduler:

```
+ (RACSignal *)interval:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler {
    return [[RACSignal interval:interval onScheduler:scheduler withLeeway:0.0] setNameWithFormat:@"+interval: %f onScheduler: %@", (double)interval, scheduler];
}
```

这个操作就是调用上一个方法interval:onScheduler:withLeeway:，只不过leeway = 0.0。具体实现上面已经分析过了，这里不再赘述。

## 过滤操作

### 1. filter: 

这个filter:操作在any:的实现中用到过了。

```
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

![](https://wtj900.github.io/img/RAC/RAC-stream-filter.png)

filter:中传入一个闭包，是用筛选的条件。如果满足筛选条件的即返回原信号的值，否则原信号的值被“吞”掉，返回空的信号。这个变换主要是用flattenMap的。

### 2. ignoreValues

```
- (RACSignal *)ignoreValues {
    return [[self filter:^(id _) {
        return NO;
    }] setNameWithFormat:@"[%@] -ignoreValues", self.name];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-ignoreValue.png)

由上面filter的实现，这里把筛选判断条件永远的传入NO，那么原信号的值都会被变换成empty信号，故变换之后的信号为空信号。

### 3. ignore:

```
- (instancetype)ignore:(id)value {
    return [[self filter:^ BOOL (id innerValue) {
        return innerValue != value && ![innerValue isEqual:value];
    }] setNameWithFormat:@"[%@] -ignore: %@", self.name, [value rac_description]];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-ignore.png)

ignore:的实现还是由filter:实现的。传入的筛选判断条件是一个值，当原信号发送的值中是这个值的时候，就替换成空信号。

### 4. distinctUntilChanged

```
- (instancetype)distinctUntilChanged {
    Class class = self.class;
 
    return [[self bind:^{
        __block id lastValue = nil;
        __block BOOL initial = YES;
 
        return ^(id x, BOOL *stop) {
            if (!initial && (lastValue == x || [x isEqual:lastValue])) return [class empty];
 
            initial = NO;
            lastValue = x;
            return [class return:x];
        };
    }] setNameWithFormat:@"[%@] -distinctUntilChanged", self.name];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-distinctUntilChanged.png)

distinctUntilChanged的实现是用bind来完成的。每次变换中都记录一下原信号上一次发送过来的值，并与这一次进行比较，如果是相同的值，就“吞”掉，返回empty信号。只有和原信号上一次发送的值不同，变换后的新信号才把这个值发送出来。

关于distinctUntilChanged，这里关注的是两两信号之间的值是否不同，有时候我们可能需要一个类似于NSSet的信号集，distinctUntilChanged就无法满足了。在ReactiveCocoa 2.5的这个版本也并没有向我们提供distinct的变换函数。

![](https://wtj900.github.io/img/RAC/RAC-stream-distinct.png)

我们可以自己实现类似的变换。实现思路也不难，可以把之前每次发送过来的信号都用数组存起来，新来的信号都去数组里面查找一遍，如果找不到，就把这个值发送出去，如果找到了，就返回empty信号。效果如上图。

### 5. take:

```
- (instancetype)take:(NSUInteger)count {
    Class class = self.class;
 
    if (count == 0) return class.empty;
 
    return [[self bind:^{
        __block NSUInteger taken = 0;
 
        return ^ id (id value, BOOL *stop) {
            if (taken < count) {
                ++taken;
                if (taken == count) *stop = YES;
                return [class return:value];
            } else {
                return nil;
            }
        };
    }] setNameWithFormat:@"[%@] -take: %lu", self.name, (unsigned long)count];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-take.png)

take:实现也非常简单，借助bind函数来实现的。入参的count是原信号取值的个数。在bind的闭包中，taken计数从0开始取原信号的值，当taken取到count个数的时候，就停止取值。

在take:的基础上我们还可以继续改造出新的变换方式。比如说，想取原信号中执行的第几个值。类似于elementAt的操作。这个操作在ReactiveCocoa 2.5的这个版本也并没有直接向我们提供出来。

![](https://wtj900.github.io/img/RAC/RAC-stream-elementAt.png)

```
// 我自己增加实现的方法
- (instancetype)elementAt:(NSUInteger)index {
    Class class = self.class;

    return [[self bind:^{
        __block NSUInteger taken = 0;

        return ^ id (id value, BOOL *stop) {
            if (index == 0) {
                *stop = YES;
                return [class return:value];
            }
            if (taken == index) {
                *stop = YES;
                return [class return:value];
            } else if (taken < index){
                taken ++;
                return [class empty];
            }else {
                return nil;
            }
        };
    }] setNameWithFormat:@"[%@] -elementAt: %lu", self.name, (unsigned long)index];
}
```

### 6. takeLast:

```
- (RACSignal *)takeLast:(NSUInteger)count {
    return [[RACSignal createSignal:^(id subscriber) {
        NSMutableArray *valuesTaken = [NSMutableArray arrayWithCapacity:count];
        return [self subscribeNext:^(id x) {
            [valuesTaken addObject:x ? : RACTupleNil.tupleNil];

            while (valuesTaken.count > count) {
                [valuesTaken removeObjectAtIndex:0];
            }
        } error:^(NSError *error) {
            [subscriber sendError:error];
        } completed:^{
            for (id value in valuesTaken) {
                [subscriber sendNext:value == RACTupleNil.tupleNil ? nil : value];
            }

            [subscriber sendCompleted];
        }];
    }] setNameWithFormat:@"[%@] -takeLast: %lu", self.name, (unsigned long)count];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-takeLast.png)

takeLast:的实现也是按照套路来。先创建一个新信号，return的时候订阅原信号。在函数内部用一个valuesTaken来保存原信号发送过来的值，原信号发多少，就存多少，直到个数溢出入参给定的count，就溢出数组第0位。这样能保证数组里面始终都装着最后count个原信号的值。

当原信号发送completed信号的时候，把数组里面存的值都sendNext出去。这里要注意的也是该变换发送信号的时机。如果原信号一直没有completed，那么takeLast:就一直没法发出任何信号来。

### 7. takeUntilBlock:

```
- (instancetype)takeUntilBlock:(BOOL (^)(id x))predicate {
    NSCParameterAssert(predicate != nil);

    Class class = self.class;

    return [[self bind:^{
        return ^ id (id value, BOOL *stop) {
            if (predicate(value)) return nil;

            return [class return:value];
        };
    }] setNameWithFormat:@"[%@] -takeUntilBlock:", self.name];
}
```

takeUntilBlock:是根据传入的predicate闭包作为筛选条件的。一旦predicate( )闭包满足条件，那么新信号停止发送新信号，因为它被置为nil了。和函数名的意思是一样的，take原信号的值，Until直到闭包满足条件。

### 8. takeWhileBlock:

```
- (instancetype)takeWhileBlock:(BOOL (^)(id x))predicate {
    NSCParameterAssert(predicate != nil);

    return [[self takeUntilBlock:^ BOOL (id x) {
        return !predicate(x);
    }] setNameWithFormat:@"[%@] -takeWhileBlock:", self.name];
}
```

takeWhileBlock:的信号集是takeUntilBlock:的信号集的补集。全集是原信号。takeWhileBlock:底层还是调用takeUntilBlock:，只不过判断条件的是不满足predicate( )闭包的集合。

### 9. takeUntil:

```
- (RACSignal *)takeUntil:(RACSignal *)signalTrigger {
    return [[RACSignal createSignal:^(id subscriber) {
        RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];
        void (^triggerCompletion)(void) = ^{
            [disposable dispose];
            [subscriber sendCompleted];
        };

        RACDisposable *triggerDisposable = [signalTrigger subscribeNext:^(id _) {
            triggerCompletion();
        } completed:^{
            triggerCompletion();
        }];

        [disposable addDisposable:triggerDisposable];

        if (!disposable.disposed) {
            RACDisposable *selfDisposable = [self subscribeNext:^(id x) {
                [subscriber sendNext:x];
            } error:^(NSError *error) {
                [subscriber sendError:error];
            } completed:^{
                [disposable dispose];
                [subscriber sendCompleted];
            }];

            [disposable addDisposable:selfDisposable];
        }

        return disposable;
    }] setNameWithFormat:@"[%@] -takeUntil: %@", self.name, signalTrigger];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-takeUntil.png)

takeUntil:的实现也是“经典套路”——return一个新信号，在新信号中订阅原信号。入参是一个信号signalTrigger，这个信号是一个Trigger。一旦signalTrigger发出第一个信号，就会触发triggerCompletion( )闭包，在这个闭包中，会调用triggerCompletion( )闭包。

一旦调用了triggerCompletion( )闭包，就会把原信号取消订阅，并给变换的新的信号订阅者sendCompleted。

如果入参signalTrigger一直没有sendNext，那么原信号就会一直sendNext:。

### 10. takeUntilReplacement:

```
- (RACSignal *)takeUntilReplacement:(RACSignal *)replacement {
    return [RACSignal createSignal:^(id subscriber) {
        RACSerialDisposable *selfDisposable = [[RACSerialDisposable alloc] init];

        RACDisposable *replacementDisposable = [replacement subscribeNext:^(id x) {
            [selfDisposable dispose];
            [subscriber sendNext:x];
        } error:^(NSError *error) {
            [selfDisposable dispose];
            [subscriber sendError:error];
        } completed:^{
            [selfDisposable dispose];
            [subscriber sendCompleted];
        }];

        if (!selfDisposable.disposed) {
            selfDisposable.disposable = [[self
                                          concat:[RACSignal never]]
                                         subscribe:subscriber];
        }

        return [RACDisposable disposableWithBlock:^{
            [selfDisposable dispose];
            [replacementDisposable dispose];
        }];
    }];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-takeUntilReplacement.png)

1. 原信号concat:了一个[RACSignal never]信号，这样原信号就一直不会disposed，会一直等待replacement信号的到来。
2. 控制selfDisposable是否被dispose，控制权来自于入参的replacement信号，一旦replacement信号sendNext，那么原信号就会取消订阅，接下来的事情就会交给replacement信号了。
3. 变换后的新信号sendNext，sendError，sendCompleted全部都由replacement信号来发送，最终新信号完成的时刻也是replacement信号完成的时刻。

### 11. skip:

```
- (instancetype)skip:(NSUInteger)skipCount {
    Class class = self.class;

    return [[self bind:^{
        __block NSUInteger skipped = 0;

        return ^(id value, BOOL *stop) {
            if (skipped >= skipCount) return [class return:value];

            skipped++;
            return class.empty;
        };
    }] setNameWithFormat:@"[%@] -skip: %lu", self.name, (unsigned long)skipCount];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-skip.png)


skip:信号集和take:信号集是补集关系，全集是原信号。take:是取原信号的前count个信号，而skip:是从原信号第count + 1位开始取信号。

skipped是一个游标，每次原信号发送一个值，就比较它和入参skipCount的大小。如果不比skipCount大，说明还需要跳过，所以就返回empty信号，否则就把原信号的值发送出来。

通过类比take系列方法，可以发现在ReactiveCocoa 2.5的这个版本也并没有向我们提供skipLast:的变换函数。这个变换函数的实现过程也不难，我们可以类比takeLast:来实现。

实现的思路也不难，原信号每次发送过来的值，都用一个数组存储起来。skipLast:是想去掉原信号最末尾的count个信号。

我们先来分析一下：假设原信号有n个信号，从0 – (n-1)，去掉最后的count个，前面还剩n – count个信号。那么从 原信号的第 count + 1位的信号开始发送，到原信号结束，这样中间就正好是发送了 n – count 个信号。

分析清楚后，代码就很容易了：

```
// 我自己增加实现的方法
- (RACSignal *)skipLast:(NSUInteger)count {
    return [[RACSignal createSignal:^(id subscriber) {
        NSMutableArray *valuesTaken = [NSMutableArray arrayWithCapacity:count];
        return [self subscribeNext:^(id x) {
            [valuesTaken addObject:x ? : RACTupleNil.tupleNil];

            while (valuesTaken.count > count) {
                [subscriber sendNext:valuesTaken[0] == RACTupleNil.tupleNil ? nil : valuesTaken[0]];
                [valuesTaken removeObjectAtIndex:0];
            }
        } error:^(NSError *error) {
            [subscriber sendError:error];
        } completed:^{            
            [subscriber sendCompleted];
        }];
    }] setNameWithFormat:@"[%@] -skipLast: %lu", self.name, (unsigned long)count];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-skipLast.png)

原信号每发送过来一个信号就存入数组，当数组里面的个数大于count的时候，就是需要我们发送信号的时候，这个时候每次都把数组里面第0位发送出去即可，数组维护了一个FIFO的队列。这样就实现了skipLast:的效果了。

### 12. skipUntilBlock:

```
- (instancetype)skipUntilBlock:(BOOL (^)(id x))predicate {
    NSCParameterAssert(predicate != nil);

    Class class = self.class;

    return [[self bind:^{
        __block BOOL skipping = YES;

        return ^ id (id value, BOOL *stop) {
            if (skipping) {
                if (predicate(value)) {
                    skipping = NO;
                } else {
                    return class.empty;
                }
            }

            return [class return:value];
        };
    }] setNameWithFormat:@"[%@] -skipUntilBlock:", self.name];
}
```

skipUntilBlock: 的实现可以类比takeUntilBlock: 的实现。

skipUntilBlock: 是根据传入的predicate闭包作为筛选条件的。一旦predicate( )闭包满足条件，那么skipping = NO。skipping为NO，以后原信号发送的每个值都原封不动的发送出去。predicate( )闭包不满足条件的时候，即会一直skip原信号的值。和函数名的意思是一样的，skip原信号的值，Until直到闭包满足条件，就不再skip了。

### 13. skipWhileBlock: 

```
- (instancetype)skipWhileBlock:(BOOL (^)(id x))predicate {
    NSCParameterAssert(predicate != nil);

    return [[self skipUntilBlock:^ BOOL (id x) {
        return !predicate(x);
    }] setNameWithFormat:@"[%@] -skipWhileBlock:", self.name];
}
```

skipWhileBlock:的信号集是skipUntilBlock:的信号集的补集。全集是原信号。skipWhileBlock:底层还是调用skipUntilBlock:，只不过判断条件的是不满足predicate( )闭包的集合。

到这里skip系列方法就结束了，对比take系列的方法，少了2个方法，在ReactiveCocoa 2.5的这个版本中 takeUntil: 和 takeUntilReplacement:这两个方法没有与之对应的skip方法。

```
// 我自己增加实现的方法
- (RACSignal *)skipUntil:(RACSignal *)signalTrigger {
    return [[RACSignal createSignal:^(id subscriber) {
        RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];

        __block BOOL sendTrigger = NO;

        void (^triggerCompletion)(void) = ^{
            sendTrigger = YES;
        };

        RACDisposable *triggerDisposable = [signalTrigger subscribeNext:^(id _) {
            triggerCompletion();
        } completed:^{
            triggerCompletion();
        }];

        [disposable addDisposable:triggerDisposable];

        if (!disposable.disposed) {
            RACDisposable *selfDisposable = [self subscribeNext:^(id x) {

                if (sendTrigger) {
                    [subscriber sendNext:x];
                }

            } error:^(NSError *error) {
                [subscriber sendError:error];
            } completed:^{
                [disposable dispose];
                [subscriber sendCompleted];
            }];

            [disposable addDisposable:selfDisposable];
        }

        return disposable;
    }] setNameWithFormat:@"[%@] -skipUntil: %@", self.name, signalTrigger];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-skipUntil.png)

skipUntil实现方法也很简单，当入参的signalTrigger开发发送信号的时候，就让原信号sendNext把值发送出来，否则就把原信号的值“吞”掉。

skipUntilReplacement:就没什么意义了，把原信号经过skipUntilReplacement:变换之后得到的新的信号就是Replacement信号。所以说这个操作也就没意义了。

### 14. groupBy:transform:

```
- (RACSignal *)groupBy:(id (^)(id object))keyBlock transform:(id (^)(id object))transformBlock {
    NSCParameterAssert(keyBlock != NULL);

    return [[RACSignal createSignal:^(id subscriber) {
        NSMutableDictionary *groups = [NSMutableDictionary dictionary];
        NSMutableArray *orderedGroups = [NSMutableArray array];

        return [self subscribeNext:^(id x) {
            id key = keyBlock(x);
            RACGroupedSignal *groupSubject = nil;
            @synchronized(groups) {
                groupSubject = groups[key];
                if (groupSubject == nil) {
                    groupSubject = [RACGroupedSignal signalWithKey:key];
                    groups[key] = groupSubject;
                    [orderedGroups addObject:groupSubject];
                    [subscriber sendNext:groupSubject];
                }
            }

            [groupSubject sendNext:transformBlock != NULL ? transformBlock(x) : x];
        } error:^(NSError *error) {
            [subscriber sendError:error];

            [orderedGroups makeObjectsPerformSelector:@selector(sendError:) withObject:error];
        } completed:^{
            [subscriber sendCompleted];

            [orderedGroups makeObjectsPerformSelector:@selector(sendCompleted)];
        }];
    }] setNameWithFormat:@"[%@] -groupBy:transform:", self.name];
}
```

看groupBy:transform:的实现，依旧是老“套路”。return 一个新的RACSignal，在新的信号里面订阅原信号。

groupBy:transform:的重点就在subscribeNext中了。

1. 首先解释一下两个入参。两个入参都是闭包，keyBlock返回值是要作为字典的key，transformBlock的返回值是对原信号发出来的值x进行变换。
2. 先创建一个NSMutableDictionary字典groups，和NSMutableArray数组orderedGroups。
3. 从字典里面取出key对应的value，这里的key对应着keyBlock返回值。value的值是一个RACGroupedSignal信号。如果找不到对应的key值，就新建一个RACGroupedSignal信号，并存入字典对应的key值，与之对应。
4. 新变换之后的信号，订阅之后，RACGroupedSignal进行sendNext，这是一个信号，如果transformBlock不为空，就发送transformBlock变换之后的值。
5. sendError和sendCompleted都要分别对数组orderedGroups里面每个RACGroupedSignal都要进行sendError或者sendCompleted。因为要对数组里面每个信号都执行一个操作，所以需要调用makeObjectsPerformSelector:withObject:方法。

经过groupBy:transform:变换之后，原信号会根据keyBlock进行分组。

写出测试代码，来看看平时应该怎么用。

```
    RACSignal *signalA = [RACSignal createSignal:^RACDisposable *(id subscriber)
                         {
                             [subscriber sendNext:@1];
                             [subscriber sendNext:@2];
                             [subscriber sendNext:@3];
                             [subscriber sendNext:@4];
                             [subscriber sendNext:@5];
                             [subscriber sendCompleted];
                             return [RACDisposable disposableWithBlock:^{
                                 NSLog(@"signal dispose");
                             }];
                         }];

    RACSignal *signalGroup = [signalA groupBy:^id(NSNumber *object) {
        return object.integerValue > 3 ? @"good" : @"bad";
    } transform:^id(NSNumber * object) {
        return @(object.integerValue * 10);
    }];

    [[[signalGroup filter:^BOOL(RACGroupedSignal *value) {
        return [(NSString *)value.key isEqualToString:@"good"];
    }] flatten]subscribeNext:^(id x) {
        NSLog(@"subscribeNext: %@", x);
    }];
```

假设原信号发送的1，2，3，4，5是代表的成绩的5个等级。当成绩大于3的都算“good”，小于3的都算“bad”。

signalGroup是原信号signalA经过groupBy:transform:得到的新的信号，这个信号是一个高阶的信号，因为它里面并不是直接装的是值，signalGroup这个信号里面装的还是信号。signalGroup里面有两个分组，分别是“good”分组和“bad”分组。

想从中取出这两个分组里面的值，需要进行一次filter:筛选。筛选之后得到对应分组的高阶信号。这时还要再进行一个flatten操作，把高阶信号变成低阶信号，再次订阅才能取到其中的值。

订阅新信号的值，输出如下：

```
subscribeNext: 40
subscribeNext: 50
```

### 15. groupBy:

```
- (RACSignal *)groupBy:(id (^)(id object))keyBlock {
    return [[self groupBy:keyBlock transform:nil] setNameWithFormat:@"[%@] -groupBy:", self.name];
}
```

groupBy:操作就是groupBy:transform:的缩减版，transform传入的为nil。

关于groupBy:可以干的事情很多，可以进行很高级的分组操作。这里可以举一个例子：

```
// 简单算法题，分离数组中的相同的元素，如果元素个数大于2，则组成一个新的数组，结果得到多个包含相同元素的数组，
    // 例如[1,2,3,1,2,3]分离成[1,1],[2,2],[3,3]
    RACSignal *signal = @[@1, @2, @3, @4,@2,@3,@3,@4,@4,@4].rac_sequence.signal;
 
      NSArray * array = [[[[signal groupBy:^NSString *(NSNumber *object) {
          return [NSString stringWithFormat:@"%@",object];
      }] map:^id(RACGroupedSignal *value) {
          return [value sequence];
      }] sequence] map:^id(RACSignalSequence * value) {
          return value.array;
      }].array;
 
    for (NSNumber * num in array) {
        NSLog(@"最后的数组%@",num);
    }
 
   // 最后输出 [1,2,3,4,2,3,3,4,4,4]变成[1],[2,2],[3,3,3],[4,4,4,4]
```

## 组合操作

### 1. startWith:

```
- (instancetype)startWith:(id)value {
 
    return [[[self.class return:value]
             concat:self]
            setNameWithFormat:@"[%@] -startWith: %@", self.name, [value rac_description]];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-startWith.png)

startWith:的实现很简单，就是先构造一个只发送一个value的信号，然后这个信号发送完毕之后接上原信号。得到的新的信号就是在原信号前面新加了一个值。

### 2. concat:(对象方法)

这里说的concat:是在父类RACStream中定义的。

```
- (instancetype)concat:(RACStream *)stream {
    return nil;
}
```

父类中定义的这个方法就返回一个nil，具体的实现还要子类去重写。

### 3. concat:(类方法)

```
+ (instancetype)concat:(id)streams {
    RACStream *result = self.empty;
    for (RACStream *stream in streams) {
        result = [result concat:stream];
    }
 
    return [result setNameWithFormat:@"+concat: %@", streams];
}
```

这个concat:后面跟着一个数组，数组里面包含这很多信号，concat:依次把这些信号concat:连接串起来。

### 4. merge:(多个信号)

```
+ (RACSignal *)merge:(id)signals {
    NSMutableArray *copiedSignals = [[NSMutableArray alloc] init];
    for (RACSignal *signal in signals) {
        [copiedSignals addObject:signal];
    }

    return [[[RACSignal
              createSignal:^ RACDisposable * (id subscriber) {
                  for (RACSignal *signal in copiedSignals) {
                      [subscriber sendNext:signal];
                  }

                  [subscriber sendCompleted];
                  return nil;
              }]
             flatten]
            setNameWithFormat:@"+merge: %@", copiedSignals];
}
```

![](https://wtj900.github.io/img/RAC/RAC-stream-merge_mul.png)

merge:后面跟一个数组。先会新建一个数组copiedSignals，把传入的信号都装到数组里。然后依次发送数组里面的信号。由于新信号也是一个高阶信号，因为sendNext会把各个信号都依次发送出去，所以需要flatten操作把这个信号转换成值发送出去。

从上图上看，上下两个信号就像被拍扁了一样，就成了新信号的发送顺序。

### 5. merge:(单个信号)

```
- (RACSignal *)merge:(RACSignal *)signal {
    return [[RACSignal
             merge:@[ self, signal ]]
            setNameWithFormat:@"[%@] -merge: %@", self.name, signal];
}
```

merge:后面参数也可以跟一个信号，那么merge:就是合并这两个信号。具体实现和merge:多个信号是一样的原理。

### 6. zip:

```
+ (instancetype)zip:(id)streams {
    return [[self join:streams block:^(RACStream *left, RACStream *right) {
        return [left zipWith:right];
    }] setNameWithFormat:@"+zip: %@", streams];
}
```

zip:后面可以跟一个数组，数组里面装的是各种信号流。

它的实现是调用了join: block: 实现的。

```
+ (instancetype)join:(id)streams block:(RACStream * (^)(id, id))block {
    RACStream *current = nil;
    // 第一步
    for (RACStream *stream in streams) {

        if (current == nil) {
            current = [stream map:^(id x) {
                return RACTuplePack(x);
            }];

            continue;
        }

        current = block(current, stream);
    }
    // 第二步
    if (current == nil) return [self empty];

    return [current map:^(RACTuple *xs) {

        NSMutableArray *values = [[NSMutableArray alloc] init];
        // 第三步
        while (xs != nil) {
            [values insertObject:xs.last ?: RACTupleNil.tupleNil atIndex:0];
            xs = (xs.count > 1 ? xs.first : nil);
        }
        // 第四步
        return [RACTuple tupleWithObjectsFromArray:values];
    }];
}
```

join: block: 的实现可以分为4步：

1. 依次打包各个信号流，把每个信号流都打包成元组RACTuple。首先第一个信号流打包成一个元组，这个元组里面就一个信号。接着把第一个元组和第二个信号执行block( )闭包里面的操作。传入的block( )闭包执行的是zipWith:的操作。这个操作是把两个信号“压”在一起。得到第二个元组，里面装着是第一个元组和第二个信号。之后每次循环都执行类似的操作，再把第二个元组和第三个信号进行zipWith:操作，以此类推下去，直到所有的信号流都循环一遍。
2. 经过第一步的循环操作之后，还是nil，那么肯定就是空信号了，就返回empty信号。
3. 这一步是把之前第一步打包出来的结果，还原回原信号的过程。经过第一步的循环之后，current会是类似这个样子，(((1), 2), 3)，第三步就是为了把这种多重元组解出来，每个信号流都依次按照顺序放在数组里。注意观察current的特点，最外层的元组，是一个值和一个元组，所以从最外层的元组开始，一层一层往里“剥”。while循环每次都取最外层元组的last，即那个单独的值，插入到数组的第0号位置，然后取出first即是里面一层的元组。然后依次循环。由于每次都插入到数组0号的位置，类似于链表的头插法，最终数组里面的顺序肯定也保证是原信号的顺序。
4. 第四步就是把还原成原信号的顺序的数组包装成元组，返回给map操作的闭包。

```
+ (instancetype)tupleWithObjectsFromArray:(NSArray *)array {
    return [self tupleWithObjectsFromArray:array convertNullsToNils:NO];
}

+ (instancetype)tupleWithObjectsFromArray:(NSArray *)array convertNullsToNils:(BOOL)convert {
    RACTuple *tuple = [[self alloc] init];

    if (convert) {
        NSMutableArray *newArray = [NSMutableArray arrayWithCapacity:array.count];
        for (id object in array) {
            [newArray addObject:(object == NSNull.null ? RACTupleNil.tupleNil : object)];
        }
        tuple.backingArray = newArray;
    } else {
        tuple.backingArray = [array copy];
    }

    return tuple;
}
```

在转换过程中，入参convertNullsToNils的含义是，是否把数组里面的NSNull转换成RACTupleNil。

这里转换传入的是NO，所以就是把数组原封不动的copy一份。

测试代码:

```
    RACSignal *signalD = [RACSignal interval:3 onScheduler:[RACScheduler mainThreadScheduler] withLeeway:0];
    RACSignal *signalO = [RACSignal interval:1 onScheduler:[RACScheduler mainThreadScheduler] withLeeway:0];
    RACSignal *signalE = [RACSignal interval:4 onScheduler:[RACScheduler mainThreadScheduler] withLeeway:0];
    RACSignal *signalB = [RACStream zip:@[signalD,signalO,signalE]];

    [signalB subscribeNext:^(id x) {
        NSLog(@"最后接收到的值 = %@",x);
    }];
```

打印输出：

```
2016-11-29 13:07:57.349 最后接收到的值 =  (
    "2016-11-29 05:07:56 +0000",
    "2016-11-29 05:07:54 +0000",
    "2016-11-29 05:07:57 +0000"
)
 
2016-11-29 13:08:01.350 最后接收到的值 =  (
    "2016-11-29 05:07:59 +0000",
    "2016-11-29 05:07:55 +0000",
    "2016-11-29 05:08:01 +0000"
)
 
2016-11-29 13:08:05.352 最后接收到的值 =  (
    "2016-11-29 05:08:02 +0000",
    "2016-11-29 05:07:56 +0000",
    "2016-11-29 05:08:05 +0000"
)
```

最后输出的信号以时间最长的为主，最后接到的信号是一个元组，里面依次包含zip:数组里每个信号在一次“压”缩周期里面的值。

### 7. zip: reduce:

```
+ (instancetype)zip:(id)streams reduce:(id (^)())reduceBlock {
    NSCParameterAssert(reduceBlock != nil);
    RACStream *result = [self zip:streams];
    if (reduceBlock != nil) result = [result reduceEach:reduceBlock];
    return [result setNameWithFormat:@"+zip: %@ reduce:", streams];
}
```

zip: reduce:是一个组合的方法。具体实现可以拆分成两部分，第一部分是先执行zip:，把数组里面的信号流依次都进行组合。这一过程的实现在上一个变换实现中分析过了。zip:完成之后，紧接着进行reduceEach:操作。

这里有一个判断reduceBlock是否为nil的判断，这个判断是针对老版本的“历史遗留问题”。在ReactiveCocoa 2.5之前的版本，是允许reduceBlock传入nil，这里为了防止崩溃，所以加上了这个reduceBlock是否为nil的判断。

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

`reduceEach:`它会动态的构造闭包，对原信号每个元组，执行reduceBlock( )闭包里面的方法。一般用法如下：

```
   [RACStream zip:@[ stringSignal, intSignal ] reduce:^(NSString *string, NSNumber *number) {
       return [NSString stringWithFormat:@"%@: %@", string, number];
   }];
```

### 8. combineLatestWith:

```
- (RACSignal *)combineLatestWith:(RACSignal *)signal {
    NSCParameterAssert(signal != nil);

    return [[RACSignal createSignal:^(id subscriber) {
        RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];

        // 初始化第一个信号的一些标志变量
        __block id lastSelfValue = nil;
        __block BOOL selfCompleted = NO;

        // 初始化第二个信号的一些标志变量
        __block id lastOtherValue = nil;
        __block BOOL otherCompleted = NO;

        // 这里是一个判断是否sendNext的闭包
        void (^sendNext)(void) = ^{ };

        // 订阅第一个信号
        RACDisposable *selfDisposable = [self subscribeNext:^(id x) { }];
        [disposable addDisposable:selfDisposable];

        // 订阅第二个信号
        RACDisposable *otherDisposable = [signal subscribeNext:^(id x) { }];
        [disposable addDisposable:otherDisposable];

        return disposable;
    }] setNameWithFormat:@"[%@] -combineLatestWith: %@", self.name, signal];
}
```

大体实现思路比较简单，在新信号里面分别订阅原信号和入参signal信号。

```
RACDisposable *selfDisposable = [self subscribeNext:^(id x) {
    @synchronized (disposable) {
        lastSelfValue = x ?: RACTupleNil.tupleNil;
        sendNext();
    }
} error:^(NSError *error) {
    [subscriber sendError:error];
} completed:^{
    @synchronized (disposable) {
        selfCompleted = YES;
        if (otherCompleted) [subscriber sendCompleted];
    }
}];
```

先来看看原信号订阅的具体实现：

在subscribeNext闭包中，记录下原信号最新发送的x值，并保存到lastSelfValue中。从此lastSelfValue变量每次都保存原信号发送过来的最新的值。然后再调用sendNext( )闭包。

在completed闭包中，selfCompleted中记录下原信号发送完成。这是还要判断otherCompleted是否完成，即入参信号signal是否发送完成，只有两者都发送完成了，组合的新信号才能算全部发送完成。

```
RACDisposable *otherDisposable = [signal subscribeNext:^(id x) {
    @synchronized (disposable) {
        lastOtherValue = x ?: RACTupleNil.tupleNil;
        sendNext();
    }
} error:^(NSError *error) {
    [subscriber sendError:error];
} completed:^{
    @synchronized (disposable) {
        otherCompleted = YES;
        if (selfCompleted) [subscriber sendCompleted];
    }
}];
```

这是对入参信号signal的处理实现。和原信号的处理方式完全一致。现在重点就要看看sendNext( )闭包中都做了些什么。

```
void (^sendNext)(void) = ^{
    @synchronized (disposable) {
        if (lastSelfValue == nil || lastOtherValue == nil) return;
        [subscriber sendNext:RACTuplePack(lastSelfValue, lastOtherValue)];
    }
};
```

在sendNext( )闭包中，如果lastSelfValue 或者 lastOtherValue 其中之一有一个为nil，就return，因为这个时候无法结合在一起。当两个信号都有值，那么就把这两个信号的最新的值打包成元组发送出来。

![](https://wtj900.github.io/img/RAC/RAC-stream-combineLatestWith.png)

可以看到，每个信号每发送出来一个新的值，都会去找另外一个信号上一个最新的值进行结合。

这里可以对比一下类似的zip:操作

![](https://wtj900.github.io/img/RAC/RAC-stream-zip.png)

zip:操作是会把新来的信号的值存起来，放在数组里，然后另外一个信号发送一个值过来就和数组第0位的值相互结合成新的元组信号发送出去，并分别移除数组里面第0位的两个值。zip:能保证每次结合的值都是唯一的，不会一个原信号的值被多次结合到新的元组信号中。但是combineLatestWith:是不能保证这一点的，在原信号或者另外一个信号新信号发送前，每次发送信号都会结合当前最新的信号，这里就会有反复结合的情况。

### 9. combineLatest:

```
+ (RACSignal *)combineLatest:(id)signals {
    return [[self join:signals block:^(RACSignal *left, RACSignal *right) {
        return [left combineLatestWith:right];
    }] setNameWithFormat:@"+combineLatest: %@", signals];
}
```

combineLatest:的实现就是把入参数组里面的每个信号都调用一次join: block:方法。传入的闭包是把两个信号combineLatestWith:一下。combineLatest:的实现就是2个操作的组合。具体实现上面也都分析过，这里不再赘述。

### 10. combineLatest: reduce:

```
+ (RACSignal *)combineLatest:(id)signals reduce:(id (^)())reduceBlock {
    NSCParameterAssert(reduceBlock != nil);
    RACSignal *result = [self combineLatest:signals];
    if (reduceBlock != nil) result = [result reduceEach:reduceBlock]; 
    return [result setNameWithFormat:@"+combineLatest: %@ reduce:", signals];
}
```

combineLatest: reduce: 的实现可以类比zip: reduce:的实现。

具体实现可以拆分成两部分，第一部分是先执行combineLatest:，把数组里面的信号流依次都进行组合。这一过程的实现在上一个变换实现中分析过了。combineLatest:完成之后，紧接着进行reduceEach:操作。

这里有一个判断reduceBlock是否为nil的判断，这个判断是针对老版本的“历史遗留问题”。在ReactiveCocoa 2.5之前的版本，是允许reduceBlock传入nil，这里为了防止崩溃，所以加上了这个reduceBlock是否为nil的判断。

### 11. combinePreviousWithStart: reduce:

这个方法的实现也是多个变换操作组合在一起的。

```
- (instancetype)combinePreviousWithStart:(id)start reduce:(id (^)(id previous, id next))reduceBlock {
    NSCParameterAssert(reduceBlock != NULL);
    return [[[self
              scanWithStart:RACTuplePack(start)
              reduce:^(RACTuple *previousTuple, id next) {
                  id value = reduceBlock(previousTuple[0], next);
                  return RACTuplePack(next, value);
              }]
             map:^(RACTuple *tuple) {
                 return tuple[1];
             }]
            setNameWithFormat:@"[%@] -combinePreviousWithStart: %@ reduce:", self.name, [start rac_description]];
}
```

combinePreviousWithStart: reduce:的实现完全可以类比scanWithStart:reduce:的实现。举个例子来说明他们俩的不同。

```
      RACSequence *numbers = @[ @1, @2, @3, @4 ].rac_sequence;

      RACSignal *signalA = [numbers combinePreviousWithStart:@0 reduce:^(NSNumber *previous, NSNumber *next) {
          return @(previous.integerValue + next.integerValue);
      }].signal;

    RACSignal *signalB = [numbers scanWithStart:@0 reduce:^(NSNumber *previous, NSNumber *next) {
        return @(previous.integerValue + next.integerValue);
    }].signal;
```

signalA输出如下：

```
1
3
5
7
```

signalB输出如下：

```
1
3
6
10
```

现在应该不同点应该很明显了。combinePreviousWithStart: reduce:实现的是两两之前的加和，而scanWithStart:reduce:实现的累加。

为什么会这样呢，具体看看combinePreviousWithStart: reduce:的实现。

虽然combinePreviousWithStart: reduce:也是调用了scanWithStart:reduce:，但是初始值是RACTuplePack(start)元组，聚合reduce的过程也有所不同：

```
id value = reduceBlock(previousTuple[0], next); 
return RACTuplePack(next, value);
```

依次调用reduceBlock( )闭包，传入previousTuple[0], next，这里reduceBlock( )闭包是进行累加的操作，所以就是把前一个元组的第0位加上后面新来的信号的值。得到的值拼成新的元组，新的元组由next和value值构成。

如果打印出上述例子中combinePreviousWithStart: reduce:的加合过程中每个信号的值，如下：

```
(
    1,
    1
)
 
 (
    2,
    3
)
 (
    3,
    5
)
 (
    4,
    7
)
```

由于这样拆成元组之后，下次再进行操作的时候，还可以拿到前一个信号的值，这样就不会形成累加的效果。

### 12. sample:

```
- (RACSignal *)sample:(RACSignal *)sampler {
    NSCParameterAssert(sampler != nil);

    return [[RACSignal createSignal:^(id subscriber) {
        NSLock *lock = [[NSLock alloc] init];
        __block id lastValue;
        __block BOOL hasValue = NO;

        RACSerialDisposable *samplerDisposable = [[RACSerialDisposable alloc] init];
        RACDisposable *sourceDisposable = [self subscribeNext:^(id x) { // 暂时省略 }];

        samplerDisposable.disposable = [sampler subscribeNext:^(id _) { // 暂时省略 }];

        return [RACDisposable disposableWithBlock:^{
            [samplerDisposable dispose];
            [sourceDisposable dispose];
        }];
    }] setNameWithFormat:@"[%@] -sample: %@", self.name, sampler];
}
```

sample:内部实现也是对原信号和入参信号sampler分别进行订阅。具体实现就是这两个信号订阅内部都干了些什么。

```
RACSerialDisposable *samplerDisposable = [[RACSerialDisposable alloc] init];
RACDisposable *sourceDisposable = [self subscribeNext:^(id x) {
    [lock lock];
    hasValue = YES;
    lastValue = x;
    [lock unlock];
} error:^(NSError *error) {
    [samplerDisposable dispose];
    [subscriber sendError:error];
} completed:^{
    [samplerDisposable dispose];
    [subscriber sendCompleted];
}];
```

这是对原信号的操作，原信号的操作在subscribeNext中就记录了两个变量的值，hasValue记录原信号有值，lastValue记录了原信号的最新的值。这里加了一层NSLock锁进行保护。

在发生error的时候，先把sampler信号取消订阅，然后再sendError:。当原信号完成的时候，同样是先把sampler信号取消订阅，然后再sendCompleted。

```
samplerDisposable.disposable = [sampler subscribeNext:^(id _) {
    BOOL shouldSend = NO;
    id value;
    [lock lock];
    shouldSend = hasValue;
    value = lastValue;
    [lock unlock];

    if (shouldSend) {
        [subscriber sendNext:value];
    }
} error:^(NSError *error) {
    [sourceDisposable dispose];
    [subscriber sendError:error];
} completed:^{
    [sourceDisposable dispose];
    [subscriber sendCompleted];
}];
```

这是对入参信号sampler的操作。shouldSend默认值是NO，这个变量控制着是否sendNext:值。只有当原信号有值的时候，hasValue = YES，所以shouldSend = YES，这个时候才能发送原信号的值。这里我们并不关心入参信号sampler的值，从subscribeNext:^(id _)这里可以看出， _代表并不需要它的值。

在发生error的时候，先把原信号取消订阅，然后再sendError:。当sampler信号完成的时候，同样是先把原信号取消订阅，然后再sendCompleted。

![](https://wtj900.github.io/img/RAC/RAC-stream-sample.png)

经过sample:变换就会变成这个样子。只是把原信号的值都移动到了sampler信号发送信号的时刻，值还是和原信号的值一样。


## 高阶信号操作

### 1. flattenMap:

flattenMap:在整个RAC中具有很重要的地位，很多信号变换都是可以用flattenMap:来实现的。

map:，flatten，filter，sequenceMany:这4个操作都是用flattenMap:来实现的。然而其他变换操作实现里面用到map:，flatten，filter又有很多。

回顾一下map:的实现：

```
- (instancetype)map:(id (^)(id value))block {
    NSCParameterAssert(block != nil);

    Class class = self.class;
    return [[self flattenMap:^(id value) {
        return [class return:block(value)];
    }] setNameWithFormat:@"[%@] -map:", self.name];
}
```

map:的操作其实就是直接原信号进行的 flattenMap:的操作，变换出来的新的信号的值是block(value)。

flatten的实现接下去会具体分析，这里先略过。

filter的实现：

```
- (instancetype)filter:(BOOL (^)(id value))block {
    NSCParameterAssert(block != nil);

    Class class = self.class;
    return [[self flattenMap:^ id (id value) {
        block(value) ? return [class return:value] :  return class.empty;
    }] setNameWithFormat:@"[%@] -filter:", self.name];
}
```

filter的实现和map:有点类似，也是对原信号进行 flattenMap:的操作，只不过block(value)不是作为返回值，而是作为判断条件，满足这个闭包的条件，变换出来的新的信号返回值就是value，不满足的就返回empty信号。

接下去要分析的高阶操作里面，switchToLatest，try:，tryMap:的实现中也将会使用到flattenMap:。

flattenMap:的源码实现：

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

flattenMap:的实现是调用了bind函数，对原信号进行变换，并返回block(value)的新信号。

从flattenMap:的源码可以看到，它是可以支持类似Promise的串行异步操作的，并且flattenMap:是满足Monad中bind部分定义的。flattenMap:没法去实现takeUntil:和take:的操作。

然而，bind操作可以实现take:的操作，bind是完全满足Monad中bind部分定义的。

### 2. flatten

flatten的源码实现：

```
- (instancetype)flatten {
    __weak RACStream *stream __attribute__((unused)) = self;
    return [[self flattenMap:^(id value) {
        return value;
    }] setNameWithFormat:@"[%@] -flatten", self.name];
}
```

flatten操作必须是对高阶信号进行操作，如果信号里面不是信号，即不是高阶信号，那么就会崩溃。崩溃信息如下：

`*** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'Value returned from -flattenMap: is not a stream`

所以flatten是对高阶信号进行的降阶操作。高阶信号每发送一次信号，经过flatten变换，由于flattenMap:操作之后，返回的新的信号的每个值就是原信号中每个信号的值。

![](https://wtj900.github.io/img/RAC/RAC-stream-flatten.png)

如果对信号A，信号B，信号C进行merge:操作，可以达到和flatten一样的效果。

`[RACSignal merge:@[signalA,signalB,signalC]];`

```
+ (RACSignal *)merge:(id)signals {
    NSMutableArray *copiedSignals = [[NSMutableArray alloc] init];
    for (RACSignal *signal in signals) {
        [copiedSignals addObject:signal];
    }
 
    return [[[RACSignal
              createSignal:^ RACDisposable * (id subscriber) {
                  for (RACSignal *signal in copiedSignals) {
                      [subscriber sendNext:signal];
                  }
 
                  [subscriber sendCompleted];
                  return nil;
              }]
             flatten]
            setNameWithFormat:@"+merge: %@", copiedSignals];
}
```

现在在回来看这段代码，copiedSignals虽然是一个NSMutableArray，但是它近似合成了一个上图中的高阶信号。然后这些信号们每发送出来一个信号就发给订阅者。整个操作如flatten的字面意思一样，压平。

### 3. flatten:

flatten:操作也必须是对高阶信号进行操作，如果信号里面不是信号，即不是高阶信号，那么就会崩溃。

flatten:的实现比较复杂，一步步的来分析：

```
- (RACSignal *)flatten:(NSUInteger)maxConcurrent {
	return [[RACSignal createSignal:^(id<RACSubscriber> subscriber) {
		RACCompoundDisposable *compoundDisposable = [[RACCompoundDisposable alloc] init];

		// Contains disposables for the currently active subscriptions.
		//
		// This should only be used while synchronized on `subscriber`.
		NSMutableArray *activeDisposables = [[NSMutableArray alloc] initWithCapacity:maxConcurrent];

		// Whether the signal-of-signals has completed yet.
		//
		// This should only be used while synchronized on `subscriber`.
		__block BOOL selfCompleted = NO;

		// Subscribes to the given signal.
		__block void (^subscribeToSignal)(RACSignal *);

		// Weak reference to the above, to avoid a leak.
		__weak __block void (^recur)(RACSignal *);

		// Sends completed to the subscriber if all signals are finished.
		//
		// This should only be used while synchronized on `subscriber`.
		void (^completeIfAllowed)(void) = ^{
			if (selfCompleted && activeDisposables.count == 0) {
				[subscriber sendCompleted];

				// A strong reference is held to `subscribeToSignal` until completion,
				// preventing it from deallocating early.
				subscribeToSignal = nil;
			}
		};

		// The signals waiting to be started.
		//
		// This array should only be used while synchronized on `subscriber`.
		NSMutableArray *queuedSignals = [NSMutableArray array];

		recur = subscribeToSignal = ^(RACSignal *signal) {
			RACSerialDisposable *serialDisposable = [[RACSerialDisposable alloc] init];

			@synchronized (subscriber) {
				[compoundDisposable addDisposable:serialDisposable];
				[activeDisposables addObject:serialDisposable];
			}

			serialDisposable.disposable = [signal subscribeNext:^(id x) {
				[subscriber sendNext:x];
			} error:^(NSError *error) {
				[subscriber sendError:error];
			} completed:^{
				__strong void (^subscribeToSignal)(RACSignal *) = recur;
				RACSignal *nextSignal;

				@synchronized (subscriber) {
					[compoundDisposable removeDisposable:serialDisposable];
					[activeDisposables removeObjectIdenticalTo:serialDisposable];

					if (queuedSignals.count == 0) {
						completeIfAllowed();
						return;
					}

					nextSignal = queuedSignals[0];
					[queuedSignals removeObjectAtIndex:0];
				}

				subscribeToSignal(nextSignal);
			}];
		};

		[compoundDisposable addDisposable:[self subscribeNext:^(RACSignal *signal) {
			if (signal == nil) return;

			NSCAssert([signal isKindOfClass:RACSignal.class], @"Expected a RACSignal, got %@", signal);

			@synchronized (subscriber) {
				if (maxConcurrent > 0 && activeDisposables.count >= maxConcurrent) {
					[queuedSignals addObject:signal];

					// If we need to wait, skip subscribing to this
					// signal.
					return;
				}
			}

			subscribeToSignal(signal);
		} error:^(NSError *error) {
			[subscriber sendError:error];
		} completed:^{
			@synchronized (subscriber) {
				selfCompleted = YES;
				completeIfAllowed();
			}
		}]];

		return compoundDisposable;
	}] setNameWithFormat:@"[%@] -flatten: %lu", self.name, (unsigned long)maxConcurrent];
}
```

先来解释一些变量，数组的作用

activeDisposables里面装的是当前正在订阅的订阅者们的disposables信号。

queuedSignals里面装的是被暂时缓存起来的信号，它们等待被订阅。

selfCompleted表示高阶信号是否Completed。

subscribeToSignal闭包的作用是订阅所给的信号。这个闭包的入参参数就是一个信号，在闭包内部订阅这个信号，并进行一些操作。

recur是对subscribeToSignal闭包的一个弱引用，防止strong-weak循环引用，在下面会分析subscribeToSignal闭包，就会明白为什么recur要用weak修饰了。

completeIfAllowed的作用是在所有信号都发送完毕的时候，通知订阅者，给订阅者发送completed。

入参maxConcurrent的意思是最大可容纳同时被订阅的信号个数。

再来详细分析一下具体订阅的过程。

flatten:的内部，订阅高阶信号发出来的信号，这部分的代码比较简单：

```
    [self subscribeNext:^(RACSignal *signal) {
        if (signal == nil) return;

        NSCAssert([signal isKindOfClass:RACSignal.class], @"Expected a RACSignal, got %@", signal);

        @synchronized (subscriber) {
            // 1
            if (maxConcurrent > 0 && activeDisposables.count >= maxConcurrent) {
                [queuedSignals addObject:signal];
                return;
            }
        }
        // 2
        subscribeToSignal(signal);
    } error:^(NSError *error) {
        [subscriber sendError:error];
    } completed:^{
        @synchronized (subscriber) {
            selfCompleted = YES;
            // 3
            completeIfAllowed();
        }
    }]];
```

1. 如果当前最大可容纳信号的个数 > 0 ，且，activeDisposables数组里面已经装满到最大可容纳信号的个数，不能再装新的信号了。那么就把当前的信号缓存到queuedSignals数组中。
2. 直到activeDisposables数组里面有空的位子可以加入新的信号，那么就调用subscribeToSignal( )闭包，开始订阅这个新的信号。
3. 最后完成的时候标记变量selfCompleted为YES，并且调用completeIfAllowed( )闭包。

```
void (^completeIfAllowed)(void) = ^{
    if (selfCompleted && activeDisposables.count == 0) {
        [subscriber sendCompleted];
        subscribeToSignal = nil;
    }
};
```

当selfCompleted = YES 并且activeDisposables数组里面的信号都发送完毕，没有可以发送的信号了，即activeDisposables.count = 0，那么就给订阅者sendCompleted。这里值得一提的是，还需要把subscribeToSignal手动置为nil。因为在subscribeToSignal闭包中强引用了completeIfAllowed闭包，防止completeIfAllowed闭包被提早的销毁掉了。所以在completeIfAllowed闭包执行完毕的时候，需要再把subscribeToSignal闭包置为nil。

那么接下来需要看的重点就是subscribeToSignal( )闭包。

```
    recur = subscribeToSignal = ^(RACSignal *signal) {
        RACSerialDisposable *serialDisposable = [[RACSerialDisposable alloc] init];
        // 1
        @synchronized (subscriber) {
            [compoundDisposable addDisposable:serialDisposable];
            [activeDisposables addObject:serialDisposable];
        }

        serialDisposable.disposable = [signal subscribeNext:^(id x) {
            [subscriber sendNext:x];
        } error:^(NSError *error) {
            [subscriber sendError:error];
        } completed:^{
            // 2
            __strong void (^subscribeToSignal)(RACSignal *) = recur;
            RACSignal *nextSignal;
            // 3
            @synchronized (subscriber) {
                [compoundDisposable removeDisposable:serialDisposable];
                [activeDisposables removeObjectIdenticalTo:serialDisposable];
                // 4
                if (queuedSignals.count == 0) {
                    completeIfAllowed();
                    return;
                }
                // 5
                nextSignal = queuedSignals[0];
                [queuedSignals removeObjectAtIndex:0];
            }
            // 6
            subscribeToSignal(nextSignal);
        }];
    };
```

1. activeDisposables先添加当前高阶信号发出来的信号的Disposable( 也就是入参信号的Disposable)
2. 这里会对recur进行__strong，因为下面第6步会用到subscribeToSignal( )闭包，同样也是为了防止出现循环引用。
3. 订阅入参信号，给订阅者发送信号。当发送完毕后，activeDisposables中移除它对应的Disposable。
4. 如果当前缓存的queuedSignals数组里面没有缓存的信号，那么就调用completeIfAllowed( )闭包。
5. 如果当前缓存的queuedSignals数组里面有缓存的信号，那么就取出第0个信号，并在queuedSignals数组移除它。
6. 把第4步取出的信号继续订阅，继续调用subscribeToSignal( )闭包。

总结一下：高阶信号每发送一个信号值，判断activeDisposables数组装的个数是否已经超过了maxConcurrent。如果装不下了就缓存进queuedSignals数组中。如果还可以装的下就开始调用subscribeToSignal( )闭包，订阅当前信号。

每发送完一个信号就判断缓存数组queuedSignals的个数，如果缓存数组里面已经没有信号了，那么就结束原来高阶信号的发送。如果缓存数组里面还有信号就继续订阅。如此循环，直到原高阶信号所有的信号都发送完毕。

整个flatten:的执行流程都分析清楚了，最后，关于入参maxConcurrent进行更进一步的解读。

回看上面flatten:的实现中有这样一句话:

`if (maxConcurrent > 0 && activeDisposables.count >= maxConcurrent)`

那么maxConcurrent的值域就是最终决定flatten:表现行为。

如果maxConcurrent

`NSMutableArray *activeDisposables = [[NSMutableArray alloc] initWithCapacity:maxConcurrent];`

activeDisposables在初始化的时候会初始化一个大小为maxConcurrent的NSMutableArray。如果maxConcurrent

如果maxConcurrent = 0，会发生什么？那么flatten:就退化成flatten了。

![](https://wtj900.github.io/img/RAC/RAC-stream-flatten_0.png)

如果maxConcurrent = 1，会发生什么？那么flatten:就退化成concat了。

![](https://wtj900.github.io/img/RAC/RAC-stream-flatten_1.png)

如果maxConcurrent > 1，会发生什么？由于至今还没有遇到能用到maxConcurrent > 1的需求情况，所以这里暂时不展示图解了。maxConcurrent > 1之后，flatten的行为还依照高阶信号的个数和maxConcurrent的关系。如果高阶信号的个数maxConcurrent的值，那么多的信号就会进入queuedSignals缓存数组。

### 4. concat

这里的concat实现是在RACSignal里面定义的。

```
- (RACSignal *)concat {
    return [[self flatten:1] setNameWithFormat:@"[%@] -concat", self.name];
}
```

一看源码就知道了，concat其实就是flatten:1。

当然在RACSignal中定义了concat:方法，这个方法在之前的文章已经分析过了，这里回顾对比一下：

```
- (RACSignal *)concat:(RACSignal *)signal {
    return [[RACSignal createSignal:^(id subscriber) {
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

![](https://wtj900.github.io/img/RAC/RAC-stream-concat.png)

经过对比可以发现，虽然最终变换出来的结果类似，但是针对的信号的对象是不同的，concat是针对高阶信号进行降阶操作。concat:是把两个信号连接起来的操作。如果把高阶信号按照时间轴，从左往右，依次把每个信号都concat:连接起来，那么结果就是concat。

### 5. switchToLatest

```
- (RACSignal *)switchToLatest {
    return [[RACSignal createSignal:^(id subscriber) {
        RACMulticastConnection *connection = [self publish];

        RACDisposable *subscriptionDisposable = [[connection.signal
                                                  flattenMap:^(RACSignal *x) {
                                                      NSCAssert(x == nil || [x isKindOfClass:RACSignal.class], @"-switchToLatest requires that the source signal (%@) send signals. Instead we got: %@", self, x);
                                                      return [x takeUntil:[connection.signal concat:[RACSignal never]]];
                                                  }]
                                                 subscribe:subscriber];

        RACDisposable *connectionDisposable = [connection connect];
        return [RACDisposable disposableWithBlock:^{
            [subscriptionDisposable dispose];
            [connectionDisposable dispose];
        }];
    }] setNameWithFormat:@"[%@] -switchToLatest", self.name];
}
```

switchToLatest这个操作只能用在高阶信号上，如果原信号里面有不是信号的值，那么就会崩溃，崩溃信息如下：

`***** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: '-switchToLatest requires that the source signal ( name: ) send signals.`

在switchToLatest操作中，先把原信号转换成热信号，connection.signal就是RACSubject类型的。对RACSubject进行flattenMap:变换。在flattenMap:变换中，connection.signal会先concat:一个never信号。这里concat:一个never信号的原因是为了内部的信号过早的结束而导致订阅者收到complete信号。

flattenMap:变换中x也是一个信号，对x进行takeUntil:变换，效果就是下一个信号到来之前，x会一直发送信号，一旦下一个信号到来，x就会被取消订阅，开始订阅新的信号。

![](https://wtj900.github.io/img/RAC/RAC-stream-switchToLatest.png)

一个高阶信号经过switchToLatest降阶操作之后，能得到上图中的信号。

### 6. switch: cases: default:

switch: cases: default:源码实现如下:

```
+ (RACSignal *)switch:(RACSignal *)signal cases:(NSDictionary *)cases default:(RACSignal *)defaultSignal {
    NSCParameterAssert(signal != nil);
    NSCParameterAssert(cases != nil);

    for (id key in cases) {
        id value __attribute__((unused)) = cases[key];
        NSCAssert([value isKindOfClass:RACSignal.class], @"Expected all cases to be RACSignals, %@ isn't", value);
    }

    NSDictionary *copy = [cases copy];

    return [[[signal
              map:^(id key) {
                  if (key == nil) key = RACTupleNil.tupleNil;

                  RACSignal *signal = copy[key] ?: defaultSignal;
                  if (signal == nil) {
                      NSString *description = [NSString stringWithFormat:NSLocalizedString(@"No matching signal found for value %@", @""), key];
                      return [RACSignal error:[NSError errorWithDomain:RACSignalErrorDomain code:RACSignalErrorNoMatchingCase userInfo:@{ NSLocalizedDescriptionKey: description }]];
                  }

                  return signal;
              }]
             switchToLatest]
            setNameWithFormat:@"+switch: %@ cases: %@ default: %@", signal, cases, defaultSignal];
}
```

实现中有3个断言，全部都是针对入参的要求。入参signal信号和cases字典都不能是nil。其次，cases字典里面所有key对应的value必须是RACSignal类型的。注意，defaultSignal是可以为nil的。

接下来的实现比较简单，对入参传进来的signal信号进行map变换，这里的变换是升阶的变换。

signal每次发送出来的一个值，就把这个值当做key值去cases字典里面去查找对应的value。当然value对应的是一个信号。如果value对应的信号不为空，就把signal发送出来的这个值map成字典里面对应的信号。如果value对应为空，那么就把原signal发出来的值map成defaultSignal信号。

如果经过转换之后，得到的信号为nil，就会返回一个error信号。如果得到的信号不为nil，那么原信号完全转换完成就会变成一个高阶信号，这个高阶信号里面装的都是信号。最后再对这个高阶信号执行switchToLatest转换。

### 7. if: then: else:

if: then: else:源码实现如下:

```
+ (RACSignal *)if:(RACSignal *)boolSignal then:(RACSignal *)trueSignal else:(RACSignal *)falseSignal {
    NSCParameterAssert(boolSignal != nil);
    NSCParameterAssert(trueSignal != nil);
    NSCParameterAssert(falseSignal != nil);

    return [[[boolSignal
              map:^(NSNumber *value) {
                  NSCAssert([value isKindOfClass:NSNumber.class], @"Expected %@ to send BOOLs, not %@", boolSignal, value);

                  return (value.boolValue ? trueSignal : falseSignal);
              }]
             switchToLatest]
            setNameWithFormat:@"+if: %@ then: %@ else: %@", boolSignal, trueSignal, falseSignal];
}
```

入参boolSignal，trueSignal，falseSignal三个信号都不能为nil。

boolSignal里面都必须装的是NSNumber类型的值。

针对boolSignal进行map升阶操作，boolSignal信号里面的值如果是YES，那么就转换成trueSignal信号，如果为NO，就转换成falseSignal。升阶转换完成之后，boolSignal就是一个高阶信号，然后再进行switchToLatest操作。

### 8. catch:

catch:的实现如下:

```
- (RACSignal *)catch:(RACSignal * (^)(NSError *error))catchBlock {
    NSCParameterAssert(catchBlock != NULL);

    return [[RACSignal createSignal:^(id subscriber) {
        RACSerialDisposable *catchDisposable = [[RACSerialDisposable alloc] init];

        RACDisposable *subscriptionDisposable = [self subscribeNext:^(id x) {
            [subscriber sendNext:x];
        } error:^(NSError *error) {
            RACSignal *signal = catchBlock(error);
            NSCAssert(signal != nil, @"Expected non-nil signal from catch block on %@", self);
            catchDisposable.disposable = [signal subscribe:subscriber];
        } completed:^{
            [subscriber sendCompleted];
        }];

        return [RACDisposable disposableWithBlock:^{
            [catchDisposable dispose];
            [subscriptionDisposable dispose];
        }];
    }] setNameWithFormat:@"[%@] -catch:", self.name];
}
```

当对原信号进行订阅的时候，如果出现了错误，会去执行catchBlock( )闭包，入参为刚刚产生的error。catchBlock( )闭包产生的是一个新的RACSignal，并再次用订阅者订阅该信号。

这里之所以说是高阶操作，是因为这里原信号发生错误之后，错误会升阶成一个信号。

### 9. catchTo:

catchTo:的实现如下：

```
- (RACSignal *)catchTo:(RACSignal *)signal {
    return [[self catch:^(NSError *error) {
        return signal;
    }] setNameWithFormat:@"[%@] -catchTo: %@", self.name, signal];
}
```

catchTo:的实现就是调用catch:方法，只不过原来catch:方法里面的catchBlock( )闭包，永远都只返回catchTo:的入参，signal信号。

### 10. try:

```
- (RACSignal *)try:(BOOL (^)(id value, NSError **errorPtr))tryBlock {
    NSCParameterAssert(tryBlock != NULL);

    return [[self flattenMap:^(id value) {
        NSError *error = nil;
        BOOL passed = tryBlock(value, &error);
        return (passed ? [RACSignal return:value] : [RACSignal error:error]);
    }] setNameWithFormat:@"[%@] -try:", self.name];
}
```

try:也是一个高阶操作。对原信号进行flattenMap变换，对信号发出来的每个值都调用一遍tryBlock( )闭包，如果这个闭包的返回值是YES，那么就返回[RACSignal return:value]，如果闭包的返回值是NO，那么就返回error。原信号中如果都是值，那么经过try:操作之后，每个值都会变成RACSignal，于是原信号也就变成了高阶信号了。

### 11. tryMap:

```
- (RACSignal *)tryMap:(id (^)(id value, NSError **errorPtr))mapBlock {
    NSCParameterAssert(mapBlock != NULL);

    return [[self flattenMap:^(id value) {
        NSError *error = nil;
        id mappedValue = mapBlock(value, &error);
        return (mappedValue == nil ? [RACSignal error:error] : [RACSignal return:mappedValue]);
    }] setNameWithFormat:@"[%@] -tryMap:", self.name];
}
```

tryMap:的实现和try:的实现基本一致，唯一不同的就是入参闭包的返回值不同。在tryMap:中调用mapBlock( )闭包，返回是一个对象，如果这个对象不为nil，就返回[RACSignal return:mappedValue]。如果返回的对象是nil，那么就变换成error信号。

### 12. timeout: onScheduler:

```
- (RACSignal *)timeout:(NSTimeInterval)interval onScheduler:(RACScheduler *)scheduler {
    NSCParameterAssert(scheduler != nil);
    NSCParameterAssert(scheduler != RACScheduler.immediateScheduler);

    return [[RACSignal createSignal:^(id subscriber) {
        RACCompoundDisposable *disposable = [RACCompoundDisposable compoundDisposable];

        RACDisposable *timeoutDisposable = [scheduler afterDelay:interval schedule:^{
            [disposable dispose];
            [subscriber sendError:[NSError errorWithDomain:RACSignalErrorDomain code:RACSignalErrorTimedOut userInfo:nil]];
        }];

        [disposable addDisposable:timeoutDisposable];

        RACDisposable *subscriptionDisposable = [self subscribeNext:^(id x) {
            [subscriber sendNext:x];
        } error:^(NSError *error) {
            [disposable dispose];
            [subscriber sendError:error];
        } completed:^{
            [disposable dispose];
            [subscriber sendCompleted];
        }];

        [disposable addDisposable:subscriptionDisposable];
        return disposable;
    }] setNameWithFormat:@"[%@] -timeout: %f onScheduler: %@", self.name, (double)interval, scheduler];
}
```

timeout: onScheduler:的实现很简单，它比正常的信号订阅多了一个timeoutDisposable操作。它在信号订阅的内部开启了一个scheduler，经过interval的时间之后，就会停止订阅原信号，并对订阅者sendError。

这个操作的表意和方法名完全一致，经过interval的时间之后，就算timeout，那么就停止订阅原信号，并sendError。

> 总结一下ReactiveCocoa v2.5中高阶信号的升阶 / 降阶操作：
> 
> > 升阶操作：
> > 
> > > 1. map( 把值map成一个信号)
> > 
> > > 2. [RACSignal return:signal]
>  
> > 降阶操作：
> > 
> > > 1. flatten(等效于flatten:0，+merge:)
> > 
> > > 2. concat(等效于flatten:1)
> > 
> > > 3. flatten:1
> > 
> > > 4. switchToLatest
> > 
> > > 5. flattenMap:
>
> 这5种操作能将高阶信号变为低阶信号，但是最终降阶之后的效果就只有3种：switchToLatest，flatten，concat。具体的图示见上面的分析。

## 同步操作

## 副作用操作

## 多线程操作

## 其他操作

![](https://wtj900.github.io/img/RAC/RAC-stream-map.png)














