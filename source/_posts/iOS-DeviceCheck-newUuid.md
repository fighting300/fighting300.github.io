---
title: iOS11 DeviceCheck-苹果的微型云服务?     
date: 2017-09-27 11:20:47
tags: iOS 11
categories: iOS
---

iOS系统一致以来都致力于保护用户的隐私安全，从UUID到VenderId，再到Mac地址，每个开发者都在探索能跟踪用户和设备的唯一标志符，可惜自从UUID别限制使用以后，到现在还没有一个类似当初UUID的标志符。

不过在iOS11中新增了DeviceCheck的framework，可以允许你通过自己的服务器与Apple服务器通讯，并为单个设备设置(per Device、per Developer)两个bit的数据(bit是计算机表示信息的最小单元，吐槽下真的很小，相当于只能保存两个bool值)。该framework为追踪用户提供了更好的选择，不过目前看来只能存储类似用户是否使用过优惠码的信息，或者当前设备是否涉嫌刷单等欺诈行为。

![DeviceCheck 2bits](http://ojca2gwha.bkt.clouddn.com/DeviceCheck-bits.png)    

#### iOS集成介绍

DeviceCheck使用的基本流程如下：  
1.在App内使用DeviceCheck APIs获取一个临时的token来标志该设备(很快会失效)  
2.将token传递给后端服务器，随后后端服务器使用该token和认证后的key值(来自Apple的证书请求模块)请求Apple服务器，来更新或查询该设备的值  
3.Apple服务器返回后端服务器信息，获取该设备之前存储的信息，包含bits信息和最后一次修改的时间戳   

![DeviceCheck 流程](http://ojca2gwha.bkt.clouddn.com/DeviceCheck-flow.png)  

在服务端，需要使用Http Post请求来查询和更新信息，每个请求对应的header应该包含认证后的来自Apple的key值(JWT，JSON web token)。要获取该token，可以使用类似Apple推送服务(APNs)的类似流程来操作，具体流程可以参考[Communicate with APNs using authentication tokens](https://help.apple.com/xcode/mac/current/#/dev54d690a66)。

服务端请求的API请参照[Accessing and modifying Per-Device Data](https://developer.apple.com/documentation/devicecheck/accessing_and_modifying_per_device_data?changes=latest_minor)，数据payload的格式如下图示例：

![服务端请求payLoad](http://ojca2gwha.bkt.clouddn.com/DeviceCheck-payload.png)

Apple API地址：
开发环境：  https://api.development.devicecheck.apple.com  
线上环境：  https://api.devicecheck.apple.com.

请求完整后就可以获取/更新最新的设备信息了，之后可以使用该信息进行业务操作。

#### API使用

目前发现该API在Swift中无法使用，可能是目前的一个bug。不过可以通过桥oc代码的方式运行在Swift项目中。代码示例如下：     

```
+ (NSString *)getNewDeviceId {
    if ([DCDevice.currentDevice isSupported]) {
        [DCDevice.currentDevice generateTokenWithCompletionHandler:^(NSData * _Nullable token, NSError * _Nullable error) {
            if (error) {
                NSLog(@"%@", error.description);
            } else {
                // upload token to APP server
                NSString *deviceToken = [token base64EncodedStringWithOptions:NSDataBase64EncodingEndLineWithLineFeed];
                NSLog(@"%lu %@", token.length, deviceToken);
            }
        }];
    }
    return @"";
}
```


##### 参考
1. https://developer.apple.com/documentation/devicecheck
2. https://developer.apple.com/documentation/devicecheck/accessing_and_modifying_per_device_data?changes=latest_minor
