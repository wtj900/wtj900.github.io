---
layout:     post
title:      AFNetworking学习之路(二)
subtitle:   处理请求和响应 AFURLSerialization
date:       2017-10-19
author:     JT
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
    - AFNetworking
---

> AFNetworking对**发出请求以及接收响应**的过程进行序列化，这涉及到两个模块：
> 
> * AFURLResponseSerialization
> * AFURLRequestSerialization
> 
> 前者是处理响应的模块，将请求返回的数据解析成对应的格式。而后者的主要作用是修改请求（主要是 HTTP 请求）的头部，提供了一些语义明确的接口设置 HTTP 头部字段。
> 
> 我们首先会对 AFURLResponseSerialization 进行简单的介绍，因为这个模块使用在 AFURLSessionManager 也就是核心类中，而后者 AFURLRequestSerialization 主要用于 AFHTTPSessionManager 中，因为它主要用于修改 HTTP 头部。


# AFURLResponseSerialization
其实在整个`AFNetworking`项目中并不存在`AFURLResponseSerialization`这个类，这只是一个协议，**遵循这个协议的类会将数据解码成更有意义的表现形式**。协议的内容也非常简单，只有一个必须实现的方法。

~~~
@protocol AFURLResponseSerialization <NSObject, NSSecureCoding, NSCopying>

- (nullable id)responseObjectForResponse:(nullable NSURLResponse *)response
                           data:(nullable NSData *)data
                          error:(NSError * _Nullable __autoreleasing *)error NS_SWIFT_NOTHROW;

@end
~~~

遵循这个协议的类同时也要遵循 NSObject、NSSecureCoding 和 NSCopying 这三个协议，实现安全编码、拷贝以及 Objective-C 对象的基本行为。


# AFHTTPResponseSerializer

下面我们对模块中最重要的根类的实现进行分析，也就是`AFHTTPResponseSerializer`。它是在`AFURLResponseSerialization`模块中最基本的类（因为`AFURLResponseSerialization`只是一个协议）

## 初始化

首先是这个类的实例化方法：

```
+ (instancetype)serializer {
    return [[self alloc] init];
}

- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.stringEncoding = NSUTF8StringEncoding;

    self.acceptableStatusCodes = [NSIndexSet indexSetWithIndexesInRange:NSMakeRange(200, 100)];
    self.acceptableContentTypes = nil;

    return self;
}
```

因为是对**HTTP**响应进行序列化，所以这里设置了`stringEncoding`为`NSUTF8StringEncoding`而且没有对接收的内容类型加以限制。
将`acceptableStatusCodes`设置为从**200**到**299**之间的状态码, 因为只有这些状态码表示获得了有效的响应。

## 验证响应的有效性

`AFHTTPResponseSerializer`中方法的实现最长，并且最重要的就是`- [AFHTTPResponseSerializer validateResponse:data:error:]`

```
- (BOOL)validateResponse:(NSHTTPURLResponse *)response
                    data:(NSData *)data
                   error:(NSError * __autoreleasing *)error
{
    BOOL responseIsValid = YES;
    NSError *validationError = nil;

    if (response && [response isKindOfClass:[NSHTTPURLResponse class]]) {
    
        // MIME (Multipurpose Internet Mail Extensions) 是描述消息内容类型的因特网标准。
        // MIME 消息能包含文本、图像、音频、视频以及其他应用程序专用的数据。
        if (self.acceptableContentTypes && ![self.acceptableContentTypes containsObject:[response MIMEType]] &&
            !([response MIMEType] == nil && [data length] == 0)) {
            #1: 返回内容类型无效
        }

        if (self.acceptableStatusCodes && ![self.acceptableStatusCodes containsIndex:(NSUInteger)response.statusCode] && [response URL]) {
            #2: 返回状态码无效
        }
    }

    if (error && !responseIsValid) {
        *error = validationError;
    }

    return responseIsValid;
}
```

这个方法根据在初始化方法中初始化的属性`acceptableContentTypes`和`acceptableStatusCodes`来判断当前响应是否有效。

```
if ([data length] > 0 && [response URL]) {
    NSMutableDictionary *mutableUserInfo = [@{
                                                          NSLocalizedDescriptionKey: [NSString stringWithFormat:NSLocalizedStringFromTable(@"Request failed: unacceptable content-type: %@", @"AFNetworking", nil), [response MIMEType]],
                                                          NSURLErrorFailingURLErrorKey:[response URL],
                                                          AFNetworkingOperationFailingURLResponseErrorKey: response,
                                                        } mutableCopy];
     if (data) {
         mutableUserInfo[AFNetworkingOperationFailingURLResponseDataErrorKey] = data;
     }

     validationError = AFErrorWithUnderlyingError([NSError errorWithDomain:AFURLResponseSerializationErrorDomain code:NSURLErrorCannotDecodeContentData userInfo:mutableUserInfo], validationError);
}

responseIsValid = NO;
```

其中第一、二部分的代码非常相似，出现错误时通过`AFErrorWithUnderlyingError`生成格式化之后的错误，最后设置`responseIsValid`。

```
NSMutableDictionary *mutableUserInfo = [@{
                                               NSLocalizedDescriptionKey: [NSString stringWithFormat:NSLocalizedStringFromTable(@"Request failed: %@ (%ld)", @"AFNetworking", nil), [NSHTTPURLResponse localizedStringForStatusCode:response.statusCode], (long)response.statusCode],
                                               NSURLErrorFailingURLErrorKey:[response URL],
                                               AFNetworkingOperationFailingURLResponseErrorKey: response,
                                       } mutableCopy];

if (data) {
     mutableUserInfo[AFNetworkingOperationFailingURLResponseDataErrorKey] = data;
}

validationError = AFErrorWithUnderlyingError([NSError errorWithDomain:AFURLResponseSerializationErrorDomain code:NSURLErrorBadServerResponse userInfo:mutableUserInfo], validationError);

responseIsValid = NO;

```
第二部分的代码就不说了，实现上都是差不多的。

## 协议的实现

首先是对 AFURLResponseSerialization 协议的实现

```
- (id)responseObjectForResponse:(NSURLResponse *)response
                           data:(NSData *)data
                          error:(NSError *__autoreleasing *)error
{
    [self validateResponse:(NSHTTPURLResponse *)response data:data error:error];

    return data;
}
```
调用上面的方法对响应进行验证，然后返回数据，实在是没什么难度。
之后对`NSSecureCoding`还有`NSCopying`协议的实现也都是大同小异，跟我们实现这些协议没什么区别，更没什么值得看的地方。

# AFJSONResponseSerializer

接下来，看一下`AFJSONResponseSerializer`这个继承自`AFHTTPResponseSerializer`类的实现。初始化方法只是在调用父类的初始化方法之后更新了`acceptableContentTypes`属性：

```
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.acceptableContentTypes = [NSSet setWithObjects:@"application/json", @"text/json", @"text/javascript", nil];

    return self;
}
```

## 协议的实现

这个类中与父类差别最大的就是对 AFURLResponseSerialization 协议的实现。

```
- (id)responseObjectForResponse:(NSURLResponse *)response
                           data:(NSData *)data
                          error:(NSError *__autoreleasing *)error
{
	#1: 验证请求

	#2: 解决一个由只包含一个空格的响应引起的 bug,序列化 JSON

	#3: 移除 JSON 中的 null

    if (error) {
        *error = AFErrorWithUnderlyingError(serializationError, *error);
    }

    return responseObject;
}
```
* 验证请求的有效性

```
if (![self validateResponse:(NSHTTPURLResponse *)response data:data error:error]) {
        if (!error || AFErrorOrUnderlyingErrorHasCodeInDomain(*error, NSURLErrorCannotDecodeContentData, AFURLResponseSerializationErrorDomain)) {
            return nil;
        }
}
``` 

* 解决一个由只包含一个空格的响应引起的 bug,序列化 JSON

```
id responseObject = nil;
NSError *serializationError = nil;
    
BOOL isSpace = [data isEqualToData:[NSData dataWithBytes:" " length:1]];
if (data.length > 0 && !isSpace) {
    responseObject = [NSJSONSerialization JSONObjectWithData:data options:self.readingOptions error:&serializationError];
} else {
    return nil;
}

```

* 移除 JSON 中的 null

```
if (self.removesKeysWithNullValues && responseObject) {
        responseObject = AFJSONObjectByRemovingKeysWithNullValues(responseObject, self.readingOptions);
}
```

其中移除 JSON 中 null 的函数 AFJSONObjectByRemovingKeysWithNullValues 是一个递归调用的函数：

```
static id AFJSONObjectByRemovingKeysWithNullValues(id JSONObject, NSJSONReadingOptions readingOptions) {
    if ([JSONObject isKindOfClass:[NSArray class]]) {
        NSMutableArray *mutableArray = [NSMutableArray arrayWithCapacity:[(NSArray *)JSONObject count]];
        for (id value in (NSArray *)JSONObject) {
            [mutableArray addObject:AFJSONObjectByRemovingKeysWithNullValues(value, readingOptions)];
        }

        return (readingOptions & NSJSONReadingMutableContainers) ? mutableArray : [NSArray arrayWithArray:mutableArray];
    } else if ([JSONObject isKindOfClass:[NSDictionary class]]) {
        NSMutableDictionary *mutableDictionary = [NSMutableDictionary dictionaryWithDictionary:JSONObject];
        for (id <NSCopying> key in [(NSDictionary *)JSONObject allKeys]) {
            id value = (NSDictionary *)JSONObject[key];
            if (!value || [value isEqual:[NSNull null]]) {
                [mutableDictionary removeObjectForKey:key];
            } else if ([value isKindOfClass:[NSArray class]] || [value isKindOfClass:[NSDictionary class]]) {
                mutableDictionary[key] = AFJSONObjectByRemovingKeysWithNullValues(value, readingOptions);
            }
        }

        return (readingOptions & NSJSONReadingMutableContainers) ? mutableDictionary : [NSDictionary dictionaryWithDictionary:mutableDictionary];
    }

    return JSONObject;
}
```

# AFURLRequestSerialization

`AFURLRequestSerialization`也是一个协议，而负责请求解析的实际上是`AFHTTPRequestSerializer`及其子类,`AFHTTPRequestSerializer`及其子类遵循协议，主要工作是对发出的`HTTP`请求进行处理，它有几部分的工作需要完成。

1. 处理查询的 URL 参数
2. 设置 HTTP 头部字段
3. 设置请求的属性
4. 分块上传

> 这篇文章不会对其中涉及分块上传的部分进行分析，因为其中涉及到了多个类的功能，比较复杂，如果有兴趣可以研究一下。

## 处理查询参数

处理查询参数这部分主要是通过`AFQueryStringPair`还有一些`C函数`来完成的，这个类有两个属性`field`和`value`对应`HTTP`请求的查询`URL`中的参数。

```
@interface AFQueryStringPair : NSObject
@property (readwrite, nonatomic, strong) id field;
@property (readwrite, nonatomic, strong) id value;

- (instancetype)initWithField:(id)field value:(id)value;

- (NSString *)URLEncodedStringValue;
@end
```

其中的 `- [AFQueryStringPair URLEncodedStringValue]` 方法会返回 `key=value` 这种格式，同时使用 `AFPercentEscapedStringFromString` 函数来对 `field` 和 `value` 进行处理，将其中的 :#[]@!$&'()*+,;= 等字符转换为百分号表示的形式。

这一部分代码还负责返回查询参数，将 `AFQueryStringPair` 或者 `key` `value` 转换为以下这种形式：

```
username=dravenss&password=123456&hello[world]=helloworld
```
它的实现主要依赖于一个递归函数 `AFQueryStringPairsFromKeyAndValue`，如果当前的 `value` 是一个集合类型的话，那么它就会不断地递归调用自己。

```
NSArray * AFQueryStringPairsFromKeyAndValue(NSString *key, id value) {
    NSMutableArray *mutableQueryStringComponents = [NSMutableArray array];

    NSSortDescriptor *sortDescriptor = [NSSortDescriptor sortDescriptorWithKey:@"description" ascending:YES selector:@selector(compare:)];

    if ([value isKindOfClass:[NSDictionary class]]) {
        NSDictionary *dictionary = value;
        // Sort dictionary keys to ensure consistent ordering in query string, which is important when deserializing potentially ambiguous sequences, such as an array of dictionaries
        for (id nestedKey in [dictionary.allKeys sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
            id nestedValue = dictionary[nestedKey];
            if (nestedValue) {
                [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue((key ? [NSString stringWithFormat:@"%@[%@]", key, nestedKey] : nestedKey), nestedValue)];
            }
        }
    } else if ([value isKindOfClass:[NSArray class]]) {
        NSArray *array = value;
        for (id nestedValue in array) {
            [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue([NSString stringWithFormat:@"%@[]", key], nestedValue)];
        }
    } else if ([value isKindOfClass:[NSSet class]]) {
        NSSet *set = value;
        for (id obj in [set sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
            [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue(key, obj)];
        }
    } else {
        [mutableQueryStringComponents addObject:[[AFQueryStringPair alloc] initWithField:key value:value]];
    }

    return mutableQueryStringComponents;
}
```

最后返回一个数组

```
[
	username=draveness,
	password=123456,
	hello[world]=helloworld
]
```
得到这个数组之后就会调用 AFQueryStringFromParameters 使用 & 来拼接它们。

```
NSString * AFQueryStringFromParameters(NSDictionary *parameters) {
    NSMutableArray *mutablePairs = [NSMutableArray array];
    for (AFQueryStringPair *pair in AFQueryStringPairsFromDictionary(parameters)) {
        [mutablePairs addObject:[pair URLEncodedStringValue]];
    }

    return [mutablePairs componentsJoinedByString:@"&"];
}
```

## 设置 HTTP 头部字段
`AFHTTPRequestSerializer` 在头文件中提供了一些属性方便我们设置 `HTTP` 头部字段。同时，在类的内部，它提供了 `- [AFHTTPRequestSerializer setValue:forHTTPHeaderField:]` 方法来设置 `HTTP` 头部，其实它的实现都是基于一个名为 `mutableHTTPRequestHeaders` 的属性的：

```
- (NSDictionary *)HTTPRequestHeaders {
    return [NSDictionary dictionaryWithDictionary:self.mutableHTTPRequestHeaders];
}

- (void)setValue:(NSString *)value
forHTTPHeaderField:(NSString *)field
{
	[self.mutableHTTPRequestHeaders setValue:value forKey:field];
}

- (NSString *)valueForHTTPHeaderField:(NSString *)field {
    return [self.mutableHTTPRequestHeaders valueForKey:field];
}
```
在设置 HTTP 头部字段时，都会存储到这个可变字典中。而当真正使用时，会用 `HTTPRequestHeaders` 这个方法，来获取对应版本的不可变字典。

到了这里，可以来分析一下，这个类是如何设置一些我们平时常用的头部字段的。首先是 User-Agent，在 AFHTTPRequestSerializer 刚刚初始化时，就会根据当前编译的平台生成一个 userAgent 字符串：

```
userAgent = [NSString stringWithFormat:@"%@/%@ (%@; iOS %@; Scale/%0.2f)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[UIDevice currentDevice] model], [[UIDevice currentDevice] systemVersion], [[UIScreen mainScreen] scale]];

[self setValue:userAgent forHTTPHeaderField:@"User-Agent"];
```

设置验证字段时，可以使用 `- [AFHTTPRequestSerializer setAuthorizationHeaderFieldWithUsername:password:]` 方法

```
- (void)setAuthorizationHeaderFieldWithUsername:(NSString *)username
                                       password:(NSString *)password
{
    NSData *basicAuthCredentials = [[NSString stringWithFormat:@"%@:%@", username, password] dataUsingEncoding:NSUTF8StringEncoding];
    NSString *base64AuthCredentials = [basicAuthCredentials base64EncodedStringWithOptions:(NSDataBase64EncodingOptions)0];
    [self setValue:[NSString stringWithFormat:@"Basic %@", base64AuthCredentials] forHTTPHeaderField:@"Authorization"];
}
```

## 设置请求的属性
还有一写 `NSURLRequest` 的属性是通过另一种方式来设置的，AFNetworking 为这些功能提供了接口

```
@property (nonatomic, assign) BOOL allowsCellularAccess;

@property (nonatomic, assign) NSURLRequestCachePolicy cachePolicy;

@property (nonatomic, assign) BOOL HTTPShouldHandleCookies;

@property (nonatomic, assign) BOOL HTTPShouldUsePipelining;

@property (nonatomic, assign) NSURLRequestNetworkServiceType networkServiceType;

@property (nonatomic, assign) NSTimeInterval timeoutInterval;
```
它们都会通过 AFHTTPRequestSerializerObservedKeyPaths 的调用而返回。

```
static NSArray * AFHTTPRequestSerializerObservedKeyPaths() {
    static NSArray *_AFHTTPRequestSerializerObservedKeyPaths = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _AFHTTPRequestSerializerObservedKeyPaths = @[NSStringFromSelector(@selector(allowsCellularAccess)), NSStringFromSelector(@selector(cachePolicy)), NSStringFromSelector(@selector(HTTPShouldHandleCookies)), NSStringFromSelector(@selector(HTTPShouldUsePipelining)), NSStringFromSelector(@selector(networkServiceType)), NSStringFromSelector(@selector(timeoutInterval))];
    });

    return _AFHTTPRequestSerializerObservedKeyPaths;
}
```

在这些属性被设置时，会触发 KVO，然后将新的属性存储在一个名为 mutableObservedChangedKeyPaths 的字典中：

```
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(__unused id)object
                        change:(NSDictionary *)change
                       context:(void *)context
{
    if (context == AFHTTPRequestSerializerObserverContext) {
        if ([change[NSKeyValueChangeNewKey] isEqual:[NSNull null]]) {
            [self.mutableObservedChangedKeyPaths removeObject:keyPath];
        } else {
            [self.mutableObservedChangedKeyPaths addObject:keyPath];
        }
    }
}
```

然后会在生成 NSURLRequest 的时候设置这些属性。

```
NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
mutableRequest.HTTPMethod = method;

for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
    if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
        [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
    }
}
```

## 工作流程

`AFHTTPRequestSerializer` 会在 `AFHTTPSessionManager` 初始化时一并初始化，这时它会根据当前系统环境预设置一些 `HTTP` 头部字段 `Accept-Language` `User-Agent`。

```
- (instancetype)init {
    self = [super init];
    if (!self) {
        return nil;
    }

    self.stringEncoding = NSUTF8StringEncoding;

    self.mutableHTTPRequestHeaders = [NSMutableDictionary dictionary];

    // Accept-Language HTTP Header; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.4
    NSMutableArray *acceptLanguagesComponents = [NSMutableArray array];
    [[NSLocale preferredLanguages] enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
        float q = 1.0f - (idx * 0.1f);
        [acceptLanguagesComponents addObject:[NSString stringWithFormat:@"%@;q=%0.1g", obj, q]];
        *stop = q <= 0.5f;
    }];
    [self setValue:[acceptLanguagesComponents componentsJoinedByString:@", "] forHTTPHeaderField:@"Accept-Language"];

    NSString *userAgent = nil;
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"
#if TARGET_OS_IOS
    // User-Agent Header; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.43
    userAgent = [NSString stringWithFormat:@"%@/%@ (%@; iOS %@; Scale/%0.2f)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[UIDevice currentDevice] model], [[UIDevice currentDevice] systemVersion], [[UIScreen mainScreen] scale]];
#elif TARGET_OS_WATCH
    // User-Agent Header; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.43
    userAgent = [NSString stringWithFormat:@"%@/%@ (%@; watchOS %@; Scale/%0.2f)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[WKInterfaceDevice currentDevice] model], [[WKInterfaceDevice currentDevice] systemVersion], [[WKInterfaceDevice currentDevice] screenScale]];
#elif defined(__MAC_OS_X_VERSION_MIN_REQUIRED)
    userAgent = [NSString stringWithFormat:@"%@/%@ (Mac OS X %@)", [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleExecutableKey] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleIdentifierKey], [[NSBundle mainBundle] infoDictionary][@"CFBundleShortVersionString"] ?: [[NSBundle mainBundle] infoDictionary][(__bridge NSString *)kCFBundleVersionKey], [[NSProcessInfo processInfo] operatingSystemVersionString]];
#endif
#pragma clang diagnostic pop
    if (userAgent) {
        if (![userAgent canBeConvertedToEncoding:NSASCIIStringEncoding]) {
            NSMutableString *mutableUserAgent = [userAgent mutableCopy];
            if (CFStringTransform((__bridge CFMutableStringRef)(mutableUserAgent), NULL, (__bridge CFStringRef)@"Any-Latin; Latin-ASCII; [:^ASCII:] Remove", false)) {
                userAgent = mutableUserAgent;
            }
        }
        [self setValue:userAgent forHTTPHeaderField:@"User-Agent"];
    }

    // HTTP Method Definitions; see http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html
    self.HTTPMethodsEncodingParametersInURI = [NSSet setWithObjects:@"GET", @"HEAD", @"DELETE", nil];

    self.mutableObservedChangedKeyPaths = [NSMutableSet set];
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self respondsToSelector:NSSelectorFromString(keyPath)]) {
            [self addObserver:self forKeyPath:keyPath options:NSKeyValueObservingOptionNew context:AFHTTPRequestSerializerObserverContext];
        }
    }

    return self;
}
```

同时它还对一些属性进行 KVO，确保它们在改变后更新 NSMutableURLRequest 中对应的属性。

在初始化之后，如果调用了 `- [AFHTTPSessionManager dataTaskWithHTTPMethod:URLString:parameters:uploadProgress:downloadProgress:success:failure:]`，就会进入 `AFHTTPRequestSerializer` 的这一方法：

```
- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(id)parameters
                                     error:(NSError *__autoreleasing *)error
{
    // 1.对参数进行检查
    NSParameterAssert(method);
    NSParameterAssert(URLString);

    NSURL *url = [NSURL URLWithString:URLString];

    NSParameterAssert(url);

    NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
    // 2.设置 HTTP 方法
    mutableRequest.HTTPMethod = method;

    // 3.通过 mutableObservedChangedKeyPaths 字典设置 NSMutableURLRequest 的属性
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
            [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
        }
    }
    // 4.设置 HTTP 头部字段和查询参数。
    mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];

	return mutableRequest;
}
```

```
- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
{
    NSParameterAssert(request);

    NSMutableURLRequest *mutableRequest = [request mutableCopy];
    
    // 1. 通过 HTTPRequestHeaders 字典设置头部字段
    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];
    
    // 2. 调用 AFQueryStringFromParameters 将参数转换为查询参数
    NSString *query = nil;
    if (parameters) {
        if (self.queryStringSerialization) {
            NSError *serializationError;
            query = self.queryStringSerialization(request, parameters, &serializationError);

            if (serializationError) {
                if (error) {
                    *error = serializationError;
                }

                return nil;
            }
        } else {
            switch (self.queryStringSerializationStyle) {
                case AFHTTPRequestQueryStringDefaultStyle:
                    query = AFQueryStringFromParameters(parameters);
                    break;
            }
        }
    }

    // 将 parameters 添加到 URL 或者 HTTP body 中,HTTPMethodsEncodingParametersInURI默认包括`GET`, `HEAD`, `DELETE`
    // 如果 HTTP 方法为 GET HEAD 或者 DELETE，也就是在初始化方法中设置的，那么参数会追加到 URL 后面。否则会被放入 HTTP body 中。
    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        if (query && query.length > 0) {
            mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
        }
    } else {
        // #2864: an empty string is a valid x-www-form-urlencoded payload
        if (!query) {
            query = @"";
        }
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
        }
        [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
    }

    return mutableRequest;
}
```
