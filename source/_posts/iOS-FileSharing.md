---
title: iOS文件共享
tags: [iOS, FileSharing]
date: 2017-01-11 15:39:17
categories: iOS
---

iOS7以来苹果支持了AirDrop，可以通过蓝牙来在设备之间传输文件，而且一直推荐使用iClound Drive、Handoff或者AirDrop来共享文件，但是iOS其实还支持通过iTunes在电脑和APP间通过拖动来共享文件，只是这项功能不常用到。下面介绍下iOS文件共享的集成流程。
注意该功能只支持iTunes9.1及之后的版本。

#### 1.开启iTunes File Sharing

要让你的App支持文件共享，你需要在App的Info.plist中配置`UIFileSharingEnabled`值为true。配置完成后在Info.plist中展示为`Application supports iTunes file sharing`，如下图所示。

![FileShare Info.plist](http://ojca2gwha.bkt.clouddn.com/FileShare-infoplist.png)

在之前的版本中可能需要在Info.plist中配置`CFBundleDisplayName`值为Bundle Display Name即`${PRODUCT_NAME}`。

![FileShare iTunes](http://ojca2gwha.bkt.clouddn.com/FileShare-itunes.png)

之后，你就可以把你需要分享的文件放在Documents文件目录下。当你通过usb连接电脑与iPhone后，iTunes会展示`File Sharing`区域。这时候，你可以在电脑和app拖动来共享数据了。
注意这时候你在iPhone设备上并无法访问其他App的这些共享数据。还有iTunes12.7开始无法在iTunes上查看当前iPhone安装的App了，小伙伴们千万别升级了。。。

在iOS11中，还可以把App的可共享Documents文件展示在默认Files应用中，要开启这项功能，需要在Info.plist中添加`LSSupportsOpeningDocumentsInPlace`值为YES。配置完成后在Info.plist中展示为`Supports opening documents in place`。当然，当你的Documents下有文件才会在Files应用中展示。

![FileShare Files](http://ojca2gwha.bkt.clouddn.com/FileShare-Files.png)

#### 2.开启AirDrop

未完待续





##### 参考文档
1. https://support.apple.com/en-us/HT201301  
2. [https://developer.apple.com/library/content/documentation/...](https://developer.apple.com/library/content/documentation/Miscellaneous/Conceptual/iPhoneOSTechOverview/CoreServicesLayer/CoreServicesLayer.html#//apple_ref/doc/uid/TP40007898-CH10-SW30)
3.
