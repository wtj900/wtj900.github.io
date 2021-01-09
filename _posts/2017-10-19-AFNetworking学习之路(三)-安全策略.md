---
layout:     post
title:      AFNetworking学习之路(三)
subtitle:   安全策略
date:       2017-10-19
author:     JT
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - AFNetworking
---

> 自 `iOS9` 发布之后，由于新特性 `App Transport Security` 的引入，在默认行为下是不能发送 `HTTP` 请求的。很多网站都在转用 `HTTPS`，而 `AFNetworking` 中的 `AFSecurityPolicy` 就是为了阻止中间人攻击，以及其它漏洞的工具。
> 
> `AFSecurityPolicy` 主要作用就是验证 `HTTPS` 请求的证书是否有效，如果 app 中有一些敏感信息或者涉及交易信息，一定要使用 `HTTPS` 来保证交易或者用户信息的安全。

#### SecCertificateRef

SecCertificateRef 是 Security.frame 框架下一个的证书引用结构体。它内部引用了一个 X.509 的证书。

> X.509 是密码学里公钥证书的格式标准。 X.509证书里含有公钥、身份信息(比如网络主机名，组织的名称或个体名称等)和签名信息（可以是证书签发机构CA的签名，也可以是自签名）。对于一份经由可信的证书签发机构签名或者可以通过其它方式验证的证书，证书的拥有者就可以用证书及相应的私钥来创建安全的通信.

#### SecKeyRef

HTTPS 中的客户端对内容进行加密，很多可逆的加密算法都有秘钥，而 SecKeyRef 就是这些秘钥抽象的结构体引用。

#### SecTrustRef

这是一个需要验证的信任对象,包含待验证的`证书(certificates)`和支持的验证`方法(policy)`等.

#### SecTrustEvaluate

证书校验函数,在函数的内部递归地从叶节点证书到根证书验证。需要验证证书本身的合法性（验证签名完整性，验证证书有效期等);验证证书颁发者的合法性（查找颁发者的证书并检查其合法性，这个过程是递归的).而递归的终止条件是证书验证过程中遇到了锚点证书(锚点证书:嵌入到操作系统中的根证书,这个根证书是权威证书颁发机构颁发的自签名证书).

上面所说的只是一般的校验方法,那么在有的客户端中,为了确定服务端返回的证书是否是自己所需要的证书,这时我们需要在客户端中导入本地证书.整个过程代码如下:

```
    //本地导入证书
    NSString *path = @"证书路径";
    NSData *certificateData = [NSData dataWithContentsOfFile:path];
    SecCertificateRef certificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData);
    NSArray *certificateArray = @[CFBridgingRelease(certificate)];
    
    SecTrustRef trust = challenge.protectionSpace.serverTrust;
    SecTrustResultType result;
    NSURLCredential *credential = nil;
    NSURLSessionAuthChallengeDisposition disposition = NSURLSessionAuthChallengePerformDefaultHandling;
    
    SecTrustSetAnchorCertificates(trust, (__bridge CFArrayRef)certificateArray);
    
    OSStatus status = SecTrustEvaluate(trust, &result);
    if (status == errSecSuccess &&
        (result == kSecTrustResultProceed ||
         result == kSecTrustResultUnspecified))
    {
        credential = [NSURLCredential credentialForTrust:trust];
        if (credential)
        {
            disposition = NSURLSessionAuthChallengeUseCredential;
        }
        else
        {
            disposition = NSURLSessionAuthChallengePerformDefaultHandling;
        }
    }
    else
    {
        disposition = NSURLSessionAuthChallengeCancelAuthenticationChallenge;
    }
    
    if (completionHandler) {
        completionHandler(disposition, credential);
    }
```

相较于一般情况处理而言,这里多了一个证书导入,还有一个`SecTrustSetAnchorCertificates(serverTrust对象, 本地证书数组)`将本地证书数组设置成需要参与验证的锚点证书.最后还是通过`SecTrustEvaluate()`方法进行校验,假如验证的数字证书是这个锚点证书的子节点，即验证的数字证书是由锚点证书对应CA或子CA签发的，或是该证书本身，则信任该证书.

#### SecTrustResultType

表示验证结果。其中 kSecTrustResultProceed表示serverTrust验证成功，且该验证得到了用户认可(例如在弹出的是否信任的alert框中选择always trust)。 kSecTrustResultUnspecified表示 serverTrust验证成功，此证书也被暗中信任了，但是用户并没有显示地决定信任该证书。 两者取其一就可以认为对serverTrust验证成功。








# AFSSLPinningMode

```
typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
    AFSSLPinningModeNone,
    AFSSLPinningModePublicKey,
    AFSSLPinningModeCertificate,
};
```
* `AFSSLPinningModeNone`在系统的信任机构列表里验证服务端返回的证书，若证书是信任机构签发的就会通过，若是自己服务器生成的证书就不会通过。
* `AFSSLPinningModePublicKey`需要客户端保存有服务端的证书拷贝，只验证证书里的公钥，不验证证书的有效期等信息。 
* `AFSSLPinningModeCertificate`需要客户端保存有服务端的证书拷贝，这里验证分两步，第一步验证证书的域名有效期等信息，第二步是对比服务端返回的证书跟客户端返回的是否一致。 

## 初始化以及设置

在使用 `AFSecurityPolicy` 验证服务端是否受到信任之前，要对其进行初始化，使用初始化方法时，主要目的是**设置验证服务器是否受信任的方式**。

```
+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode {
    return [self policyWithPinningMode:pinningMode withPinnedCertificates:[self defaultPinnedCertificates]];
}

+ (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode withPinnedCertificates:(NSSet *)pinnedCertificates {
    AFSecurityPolicy *securityPolicy = [[self alloc] init];
    securityPolicy.SSLPinningMode = pinningMode;

    [securityPolicy setPinnedCertificates:pinnedCertificates];

    return securityPolicy;
}
```
在调用 `pinnedCertificate` 的 `setter` 方法时，会从全部的证书中取出公钥保存到 `pinnedPublicKeys` 属性中。

```
- (void)setPinnedCertificates:(NSSet *)pinnedCertificates {
    _pinnedCertificates = pinnedCertificates;

    if (self.pinnedCertificates) {
        NSMutableSet *mutablePinnedPublicKeys = [NSMutableSet setWithCapacity:[self.pinnedCertificates count]];
        for (NSData *certificate in self.pinnedCertificates) {
            id publicKey = AFPublicKeyForCertificate(certificate);
            if (!publicKey) {
                continue;
            }
            [mutablePinnedPublicKeys addObject:publicKey];
        }
        self.pinnedPublicKeys = [NSSet setWithSet:mutablePinnedPublicKeys];
    } else {
        self.pinnedPublicKeys = nil;
    }
}
```
在这里调用了 `AFPublicKeyForCertificate` 对证书进行操作，返回一个公钥。

## 操作 SecTrustRef

对 `serverTrust` 的操作的函数基本上都是 `C` 的 `API`，都定义在 `Security` 模块中，先来分析一下 `AFPublicKeyForCertificate` 的实现

```
static id AFPublicKeyForCertificate(NSData *certificate) {
    id allowedPublicKey = nil;
    SecCertificateRef allowedCertificate;
    SecCertificateRef allowedCertificates[1];
    CFArrayRef tempCertificates = nil;
    SecPolicyRef policy = nil;
    SecTrustRef allowedTrust = nil;
    SecTrustResultType result;

    allowedCertificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificate);
    __Require_Quiet(allowedCertificate != NULL, _out);

    allowedCertificates[0] = allowedCertificate;
    tempCertificates = CFArrayCreate(NULL, (const void **)allowedCertificates, 1, NULL);

    policy = SecPolicyCreateBasicX509();
    __Require_noErr_Quiet(SecTrustCreateWithCertificates(tempCertificates, policy, &allowedTrust), _out);
    __Require_noErr_Quiet(SecTrustEvaluate(allowedTrust, &result), _out);

    allowedPublicKey = (__bridge_transfer id)SecTrustCopyPublicKey(allowedTrust);

_out:
    if (allowedTrust) {
        CFRelease(allowedTrust);
    }

    if (policy) {
        CFRelease(policy);
    }

    if (tempCertificates) {
        CFRelease(tempCertificates);
    }

    if (allowedCertificate) {
        CFRelease(allowedCertificate);
    }

    return allowedPublicKey;
}
```
1、 初始化一坨临时变量

```
 id allowedPublicKey = nil;
 SecCertificateRef allowedCertificate;
 SecCertificateRef allowedCertificates[1];
 CFArrayRef tempCertificates = nil;
 SecPolicyRef policy = nil;
 SecTrustRef allowedTrust = nil;
 SecTrustResultType result;
```

2、使用 `SecCertificateCreateWithData` 通过 `DER` 表示的数据生成一个 `SecCertificateRef`，然后判断返回值是否为 `NULL`

这里使用了一个非常神奇的宏 `__Require_Quiet`，它会判断 `allowedCertificate != NULL` 是否成立，如果 `allowedCertificate` 为空就会跳到 `_out` 标签处继续执行

```
allowedCertificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificate);
 __Require_Quiet(allowedCertificate != NULL, _out);
``` 
3、通过上面的 `allowedCertificate` 创建一个 CFArray

```
allowedCertificates[0] = allowedCertificate;
 tempCertificates = CFArrayCreate(NULL, (const void **)allowedCertificates, 1, NULL);
```
4、 创建一个默认的符合 `X509` 标准的 `SecPolicyRef`，通过默认的 `SecPolicyRef` 和证书创建一个 `SecTrustRef` 用于信任评估，对该对象进行信任评估，确认生成的 `SecTrustRef` 是值得信任的。

```
 policy = SecPolicyCreateBasicX509();
 __Require_noErr_Quiet(SecTrustCreateWithCertificates(tempCertificates, policy, &allowedTrust), _out);
 __Require_noErr_Quiet(SecTrustEvaluate(allowedTrust, &result), _out);
```
5、获取公钥

```
allowedPublicKey = (__bridge_transfer id)SecTrustCopyPublicKey(allowedTrust);
```
6、释放各种 C 语言指针

> 每一个 SecTrustRef 的对象都是包含多个 SecCertificateRef 和 SecPolicyRef。其中 SecCertificateRef 可以使用 DER 进行表示，并且其中存储着公钥信息。

对它的操作还有 `AFCertificateTrustChainForServerTrust` 和 `AFPublicKeyTrustChainForServerTrust` 但是它们几乎调用了相同的 API。

```
static NSArray * AFCertificateTrustChainForServerTrust(SecTrustRef serverTrust) {
    CFIndex certificateCount = SecTrustGetCertificateCount(serverTrust);
    NSMutableArray *trustChain = [NSMutableArray arrayWithCapacity:(NSUInteger)certificateCount];

    for (CFIndex i = 0; i < certificateCount; i++) {
        SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);
        [trustChain addObject:(__bridge_transfer NSData *)SecCertificateCopyData(certificate)];
    }

    return [NSArray arrayWithArray:trustChain];
}
```

* `SecTrustGetCertificateAtIndex` 获取 `SecTrustRef` 中的证书
* `SecCertificateCopyData` 从证书中或者 `DER` 表示的数据

## 验证服务端是否受信

验证服务端是否守信是通过 `- [AFSecurityPolicy evaluateServerTrust:forDomain:]` 方法进行的。

```
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain
{

	#1: 不能隐式地信任自己签发的证书

	#2: 设置 policy

	#3: 验证证书是否有效

	#4: 根据 SSLPinningMode 对服务端进行验证

    return NO;
}
```

1、不能隐式地信任自己签发的证书

```
if (domain && self.allowInvalidCertificates && self.validatesDomainName && (self.SSLPinningMode == AFSSLPinningModeNone || [self.pinnedCertificates count] == 0)) {
        return NO;
    }
```
所以如果没有提供证书或者不验证证书，并且还设置 allowInvalidCertificates 为真，满足上面的所有条件，说明这次的验证是不安全的，会直接返回 NO

2、设置 policy

```
NSMutableArray *policies = [NSMutableArray array];
    if (self.validatesDomainName) {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
    } else {
        [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
    }
```
如果要验证域名的话，就以域名为参数创建一个 `SecPolicyRef`，否则会创建一个符合 `X509` 标准的默认 `SecPolicyRef` 对象

3、验证证书的有效性

```
if (self.SSLPinningMode == AFSSLPinningModeNone) {
    return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
} else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
    return NO;
}
```

* 如果只根据信任列表中的证书进行验证，即 self.SSLPinningMode == AFSSLPinningModeNone。如果允许无效的证书的就会直接返回 YES。不允许就会对服务端信任进行验证。
* 如果服务器信任无效，并且不允许无效证书，就会返回 NO

4、根据 SSLPinningMode 对服务器信任进行验证

```
switch (self.SSLPinningMode) {
        case AFSSLPinningModeNone:
        default:
            return NO;
        case AFSSLPinningModeCertificate: {
            ...
        }
        case AFSSLPinningModePublicKey: {
            ...
        }
    }
```
* AFSSLPinningModeNone 直接返回 NO
* AFSSLPinningModeCertificate

```
NSMutableArray *pinnedCertificates = [NSMutableArray array];
            for (NSData *certificateData in self.pinnedCertificates) {
                [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
            }
            SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);

            if (!AFServerTrustIsValid(serverTrust)) {
                return NO;
            }

            // obtain the chain after being validated, which *should* contain the pinned certificate in the last position (if it's the Root CA)
            NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);
            
            for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
                if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
                    return YES;
                }
            }
            
            return NO;
```

1. 从 `self.pinnedCertificates` 中获取 DER 表示的数据
2. 使用 `SecTrustSetAnchorCertificates` 为服务器信任设置证书
3. 判断服务器信任的有效性
4. 使用 `AFCertificateTrustChainForServerTrust` 获取服务器信任中的全部 DER 表示的证书
5. 如果 `pinnedCertificates` 中有相同的证书，就会返回 `YES`

* AFSSLPinningModePublicKey

```
NSUInteger trustedPublicKeyCount = 0;
            NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);

            for (id trustChainPublicKey in publicKeys) {
                for (id pinnedPublicKey in self.pinnedPublicKeys) {
                    if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
                        trustedPublicKeyCount += 1;
                    }
                }
            }
            return trustedPublicKeyCount > 0;
```
这部分的实现和上面的差不多，区别有两点

* 会从服务器信任中获取公钥
* pinnedPublicKeys 中的公钥与服务器信任中的公钥相同的数量大于 0，就会返回真

## 与 AFURLSessionManager 协作

在代理协议 `- URLSession:didReceiveChallenge:completionHandler: 或者 - URLSession:task:didReceiveChallenge:completionHandler:` 代理方法被调用时会运行这段代码

```
if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
    if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
        disposition = NSURLSessionAuthChallengeUseCredential;
        credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
    } else {
        disposition = NSURLSessionAuthChallengeRejectProtectionSpace;
    }
} else {
    disposition = NSURLSessionAuthChallengePerformDefaultHandling;
}
```

`NSURLAuthenticationChallenge` 表示一个认证的挑战，提供了关于这次认证的全部信息。它有一个非常重要的属性 `protectionSpace`，这里保存了需要认证的保护空间, 每一个 `NSURLProtectionSpace` 对象都保存了主机地址，端口和认证方法等重要信息。
在上面的方法中，如果保护空间中的认证方法为 `NSURLAuthenticationMethodServerTrust`，那么就会使用在上一小节中提到的方法 - `[AFSecurityPolicy evaluateServerTrust:forDomain:]` 对保护空间中的 `serverTrust` 以及域名 `host` 进行认证
根据认证的结果，会在 `completionHandler` 中传入不同的 `disposition` 和 `credential` 参数。


