---
title: iOS11最新适配指南
date: 2017-09-14 11:47:19
tags:
---

新版iPhone X发布前，就隐隐感觉到一大波工作要袭来的赶脚，果然不出所料，下面简单分享下在工作中发现的适配注意点。  
适配工作主要在UI方面，后续发现的适配点会陆续补充到该文档中。


#### 版本判断

```
  if (@available(iOS 11.0, *)) {
    // 版本适配
  }
  // 或者
  #ifdef __IPHONE_11_0   
  #endif
```

iPhoneX机型判断

```
  if (UIScreen.mainScreen.bounds.size.height == 812) {
      NSLog(@"this is iPhone X");
  }
```
#### UI适配  
##### 安全区域适配

经过试验，如果你的app之前未用LaunchScreen.Storyboard作为启动页面、需要使用LaunchScreen来当做入场页面，这样APP才会自动适配为iPhoneX的大小，但是个人感觉还有别的解决方案，找到之后会在此更新。

如果你的app用了自定义的NavigationBar，则可以直接设置additionalSafeAreaInsets来适配界面。

```
  //self.automaticallyAdjustsScrollViewInsets = true;
  self.additionalSafeAreaInsets = UIEdgeInsetsMake(-44, 0, -20, 0);
```

iPhone X由于多了大圆角、传感器(齐刘海)以及底部访问主屏幕的指示遮挡，所以需要注意原有这部分内容的设计。
<!--more-->
可能有部分APP使用了RN来实现页面，则需要在RN中修改相应NaviBar和TabBar的宽度。

##### UIScrollView & UITableView

由于iOS 11废弃了UIViewController的automaticallyAdjustsScrollViewInsets属性，所以当超出安全区域时系统自动调整了SafeAreaInsets，进而影响了adjustedContentInset，在iOS 11中决定tableView 内容与边缘距离的是adjustedContentInset.
所以需要设置UIScrollView的contentInsetAdjustmentBehavior属性。

1. UIScrollViewContentInsetAdjustmentAutomatic：如果scrollview在一个automaticallyAdjustsScrollViewContentInset = YES的controller上，并且这个Controller包含在一个navigation controller中，这种情况下会设置在top & bottom上 adjustedContentInset = safeAreaInset + contentInset不管是否滚动。其他情况下与UIScrollViewContentInsetAdjustmentScrollableAxes相同

2. UIScrollViewContentInsetAdjustmentScrollableAxes: 在可滚动方向上adjustedContentInset = safeAreaInset + contentInset，在不可滚动方向上adjustedContentInset = contentInset；依赖于scrollEnabled和alwaysBounceHorizontal / vertical = YES，scrollEnabled默认为yes，所以大多数情况下，计算方式还是adjustedContentInset = safeAreaInset + contentInset

3. UIScrollViewContentInsetAdjustmentNever: adjustedContentInset = contentInset

4. UIScrollViewContentInsetAdjustmentAlways: adjustedContentInset = safeAreaInset + contentInset


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
