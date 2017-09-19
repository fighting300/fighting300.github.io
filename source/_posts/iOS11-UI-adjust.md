---
title: iOS11/iPhoneX最新适配指南
date: 2017-09-14 11:47:19
tags: iOS 11
---

新版iPhone发布会前，就隐隐感觉到一波适配工作要袭来的赶脚，果然不出所料，下面简单分享下在整理过程中发现的适配注意点。(适配工作主要在UI方面，后续发现的适配点会陆续补充到该文档中)

#### 版本判断API

iOS11 版本判断  
```
  if (@available(iOS 11.0, *)) {
    // 版本适配
  }
  // 或者
  #ifdef __IPHONE_11_0   
  #endif
```

目前没发现有iPhoneX的机型判断API，目前可以使用size来做代替判断。  

```
  if (UIScreen.mainScreen.bounds.size.height == 812) {
      NSLog(@"this is iPhone X");
  }
```  

#### UI适配  

如果之前的APP在iPhoneX屏幕没填充满，上下有黑色区域，应该是你的app之前未用LaunchScreen.Storyboard作为启动页面，可以使用LaunchScreen来当做入场页面，这样APP才会自动适配为iPhoneX的大小。或者修改Assets中的LaunchImage，添加iPhoneX的尺寸图如下(1125*2436)。

![LuanchImage适配](http://ojca2gwha.bkt.clouddn.com/iOS11-adjust-launch.png)  

##### 导航栏适配  

iPhoneX由于多了大圆角、传感器(齐刘海)以及底部访问主屏幕的指示遮挡，所以需要注意原有这部分内容的设计。
iOS11前导航栏的高度是64，其中statusBar的高度为20，而iPhoneX的statusBar高度变为了44，如果是自定义的NaviBar，这部分需要做相应的适配。
可能有部分APP使用了RN来实现页面，不要忘了在RN中修改相应NaviBar的高度。

<!--more-->

##### 安全区域适配

iPhoneX的底部由于安全区域的原因tabBar的高度由49变为83，所以自定义的底部TabBar也需要需改其适配方案。
如果你的app用了系统的NavigationBar，则可以直接设置additionalSafeAreaInsets来适配界面。
```
  self.additionalSafeAreaInsets = UIEdgeInsetsMake(-44, 0, -34, 0);
```

##### UIScrollView & UITableView

测试过程中发现tableView会有20pt/64pt的偏移，其原因是由于iOS 11废弃了UIViewController的automaticallyAdjustsScrollViewInsets属性，新增了contentInsetAdjustmentBehavior属性，所以当超出安全区域时系统自动调整了SafeAreaInsets，进而影响了adjustedContentInset，在iOS11中决定tableView内容与边缘距离的是adjustedContentInset，所以需要设置UIScrollView的contentInsetAdjustmentBehavior属性。
可以直接使用以下代码做适配：

```
  #ifdef __IPHONE_11_0   
  if ([tableView respondsToSelector:@selector(setContentInsetAdjustmentBehavior:)]) {
    tableView.contentInsetAdjustmentBehavior = UIScrollViewContentInsetAdjustmentNever;
  }
  #endif
```

在适配过程中发现UITableView会在Header／Footer返回size为负值的情况下会(之前遗漏的bug)崩溃，这块可以自查下，而iOS11之前的版本不会。

![iPhoneX尺寸](http://ojca2gwha.bkt.clouddn.com/iOS11-iPhoneX.png)

另外有人对iPhoneX整个UIWindow做了内容的调整，只是UI还是有点丑，感兴趣的同学可以去看看[该GitHub](https://github.com/HarshilShah/NotchKit);

#### API适配

##### LocalAuthentication 本地认证    

本地认证框架提供了从具有指定安全策略(密码或生物学特征)的用户请求身份验证的功能。例如，要求用户仅使用Face ID或Touch ID进行身份验证，可使用以下代码：  

```
  let myContext = LAContext()
  let myLocalizedReasonString = <#String explaining why app needs authentication#>

  var authError: NSError?
  if #available(iOS 8.0, macOS 10.12.1, *) {
      if myContext.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &authError) {
          myContext.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, localizedReason: myLocalizedReasonString) { success, evaluateError in
              if success {
                  // 用户验证通过
              } else {
                  // 用户验证失败，处理失败信息
              }
          }
      } else {
          // 不能执行策略验证，处理验证错误信息
      }
  } else {
      // Fallback on earlier versions
  }

```

LAContext新增API如下：  
1. biometryType属性返回当前设备支持的生物学特征验证方式，他的值可以分别为typeFaceID、typeTouchID或者none。  
2. localizedReason需要验证时展示在弹框上的提示信息


##### 参考文档  
1. https://developer.apple.com/videos/fall2017/
2. https://developer.apple.com/iphone/


##### Tips
iPhone X 侧边按钮的使用方式：  
- 按一下锁屏；
- 按两下 Apple Pay；
- 按三下辅助功能快捷键（比如 VoiceOver）；
- 按五下 SOS；
- 短按 Siri；
- 长按关机；
- 按一下+Volume Up 截屏。
