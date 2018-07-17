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

# 快速搭建ReactNative跨平台开发

ReactNative可以进行跨平台开发iOS和安卓，Mac搭建简单如下:

* 安装HomeBrew
* 安装Node.js
* 安装watchman 和 flow
* 安装React Native CLI

1、 安装HomeBrew

这是安装包管理器，安装后可以方便后续安装包的安装，[HomeBrew官网安装命令](https://brew.sh)

终端命令:

`/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`

2、 安装Node.js

直接使用HomeBrew安装,可尝试使用终端命:

`brew install node`

3、 安装watchman 和 flow

watchman是React修改source文件的一个工具

安装终端命令 `brew install watchman`

flow是一个JavaScript 的静态类型检查器

安装终端命令 `brew install flow`

关于`brew update` && `brew upgrade`命令

官方文档建议定期运行`brew update` && `brew upgrade`来使您的应用程序保持最新状态。

4、 安装React Native CLI

这是个用来开发React Native的命令行工具

安装终端命令： `npm install -g react-native-cli`

哈哈~大功告成，来一个跨平台项目.

在当前目录新建一个项目

`react-native init HelloReactNative`

完成之后打开这一个项目,同时看到了android和ios

5、 管理React Native库的版本

查看本地的React Native的版本

`react-native --version`

更新本地的React Native的版本

`npm update -g react-native-cli`

查询react-native的npm包最新版本

`npm info react-native`

升级或者降级npm包的版本

`npm install --save react-native@0.18`

# 安装组件依赖库并导入

```
npm install react-native-tab-navigator --save  // 安装命令
import TabNavigator from 'react-native-tab-navigator';   // 导入命令
```

# React Native发布APP之打包iOS应用

### 第一步：导出js bundle包和图片资源

和打包React Native Android应用不同的是，我们无法通过命令一步进行导出React Native iOS应用。我们需要将JS部分的代码和图片资源等打包导出，然后通过XCode将其添加到iOS项目中。

#### 导出js bundle的命令

在React Native项目的根目录下执行：

`react-native bundle --entry-file index.ios.js --platform ios --dev false --bundle-output release_ios/main.jsbundle --assets-dest release_ios/`

![](https://wtj900.github.io/img/RN/打包命令.png)

通过上述命令，我们可以将JS部分的代码和图片资源等打包导出到release_ios目录下：

![](https://wtj900.github.io/img/RN/打包资源.png)

生成jsbundle

其中，assets为项目中的JS部分所用到的图片资源(不包括原生模块中的图片资源)，main.jsbundle是JS部分的代码。

在执行打包命令之前，我们需要先确保在我们项目的根目录有release_ios文件夹，没有的话创建一个。

#### 第二步：将js bundle包和图片资源导入到iOS项目中

这一步我们需要用到XCode，选择assets文件夹与main.jsbundle文件将其拖拽到XCode的项目导航面板中即可。

![](https://wtj900.github.io/img/RN/导入资源.png)

导入jsbundle

然后，修改AppDelegate.m文件，添加如下代码：

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
 
  NSURL *jsCodeLocation;
 //jsCodeLocation = [NSURL URLWithString:@"http://localhost:8081/index.ios.bundle?platform=ios&dev=true"];
 jsCodeLocation = [[NSBundle mainBundle] URLForResource:@"main" withExtension:@"jsbundle"];
...
  return YES;
}
```

上述代码的作用是让React Native去使用我们刚才导入的jsbundle，这样以来我们就摆脱了对本地nodejs服务器的依赖。

到目前为止呢，我们已经将js bundle包和图片资源导入到iOS项目中，接下来我们就可以发布我们的iOS应用了。

#### 第三步：发布iOS应用

之后就是正常的打包流程。