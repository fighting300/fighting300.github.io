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
```
#### UI适配  
##### 安全区域适配

经过实验，如果你的app之前未用LaunchScreen.Storyboard作为启动页面、需要使用LaunchScreen来当做入场页面，这样APP才会自动适配为iPhoneX的大小，但是个人感觉还有别的解决方案，找到之后会在此更新。


##### UITableView



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
