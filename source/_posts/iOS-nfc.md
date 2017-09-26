---
title: iOS Core NFC指南
date: 2017-08-22 19:41:17
tags:
  - iOS
  - Core NFC
categories: iOS
---

大家可能听过NFC这项功能，或者有可能你每天都在使用这个功能，比如当你在进出地铁时闸机扫描地铁卡就用到了NFC技术。
简单来说NFC就是可以让智能手机的NFC模块，可以像读卡器一般，读取电子标签的相关信息，实现NFC手机之间的数据交互或是读取其他IC卡内的数据。NFC(机场通讯)，其实由非接触式射频识别（RFID）演变而来，是一种短距高频的无线电技术，在13.56MHz频率运行于20厘米距离内。它的传输速度有106 Kbit/秒、212 Kbit/秒或者424 Kbit/秒三种。目前NFC已通过成为ISO/IEC IS 18092国际标准、ECMA-340标准与ETSI TS 102 190标准。NFC可以采用主动和被动两种读取模式。

![NFC使用场景](http://upload-images.jianshu.io/upload_images/1944396-b31bb992a4bbb9f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)  


#### Core NFC介绍  
或者你可能还吐槽过Apple怎么还不支持NFC呢，其实iPhone6已经有NFC硬件了，已支持Apple Pay支付系统，只是接口没开放，终于在今年的WWDC，苹果在iOS11系统上对开发者开放了NFC接口框架Core NFC，虽然目前权限只有只读模式。  
Apple的Core NFC可以用于检测NFC(近场通讯)标签和读取包含NDEF(NFC Data Exchange Format)数据1到5类型的标签信息，只是该功能只支持iPhone 7和iPhone 7P及以上的机型。目前Core NFC其实同时有NFC和RFID的API存在，但是RFID可能没有很高的安全性，所以苹果没有推广使用(或者还在开发中)。

> NFC Data Exchange Format : NFC数据交换格式，NFC组织约定的NFC tag中的数据格式。NDEF是轻量级的紧凑的二进制格式，可带有URL、vCard和NFC定义的各种数据类型。NDEF的由各种数据记录组成，而各个记录由报头(Header)和有效载荷(Payload)组成，其中NDEF记录的数据类型和大小由记录载荷的报头注明，这里的报头包含3部分，分别为Length、Type和Identifier。

![NFC标签图例](http://ojca2gwha.bkt.clouddn.com/CoreNFC-tags.png)

<!--more-->

#### iOS集成开发

##### 项目配置

首先需要让你的AppID添加对NFC的支持，选中NFC Tag Reading后更新Provisioning Profiles即可。

![NFC Profiles](http://ojca2gwha.bkt.clouddn.com/CoreNFC-profiles.png)

其次在项目中打开Targets->Capabilities下的Near Field Communication Tag Reading选项，Xcode会自动帮你创建NFC entitlement文件。然后你需要在entitlements文件下添加如下内容：
```
  <key>com.apple.developer.nfc.readersession.formats</key>
  <array>
    <string>NDEF</string>
  </array>
```

![NFC Capabilities](http://ojca2gwha.bkt.clouddn.com/CoreNFC-Capacities.png)

随后需要在info.plist中添加隐私标签`Privacy - NFC Scan Usage Description`。

```
  <key>NFCReaderUsageDescription</key>
  <string>NFC Import</string>
```

![NFC Info](http://ojca2gwha.bkt.clouddn.com/CoreNFC-plist.png)

##### 集成Core NFC

集成Core NFC需要用到`NFCNDEFReaderSession`类，其为NFCReaderSession的子类，但是基类不能实例化。
和iOS的其他Session一样通过其协议`NFCReaderSessionProtocol`方法来处理信息回调的结果。这最重要的一个代理回调是`func readerSession(_ session: NFCNDEFReaderSession, didDetectNDEFs messages:[NFCNDEFMessage]) `方法，用以处理检测到的NDEF消息，messages是一个NFCNDEFMessage的数组，其有一个records数组，包含NFCNDEFPayload对象，该对象存放了真正的数据内容。


```
import CoreNFC
class MessagesTableViewController: UITableViewController, NFCNDEFReaderSessionDelegate {
  // MARK: NFCNDEReaderSessionDelegate
  func readerSession(_ session: NFCNDEFReaderSession, didInvalidateWithError error: Error) {
     // Check invalidation reason from the returned error. Session will be invalidated after the function returns. New session instance is required to restart tag scanning.
  }
  func readerSession(_ session: NFCNDEFReaderSession, didDetectNDEFs messages:[NFCNDEFMessage]) {
     // Process read NFCNDEFMessage objects.
     for message in messages {
         print(" - \(message.records.count) Records:")
         for record in message.records {
             print("\t- TNF (TypeNameFormat): \(record.typeNameFormat)")
             print("\t- Payload: \(String(data: record.payload, encoding: .utf8)!)")
             print("\t- Type: \(record.type)")
             print("\t- Identifier: \(record.identifier)\n")
         }
      }
  }

  // MARK: - Actions
  @IBAction func beginScanning(_ sender: Any) {
     let session = NFCNDEFReaderSession(delegate: self, queue: nil, invalidatedAfterFirstRead:true)
     session.alertMessage = "You can scan NFC-tags by holding them behind the top of your iPhone."
     session.begin()
  }
```

`NFCReaderSession`还有`NFCISO15693ReaderSession`的子类，用于RFID的读取处理，其使用流程和`NFCNDEFReaderSession`基本一致，只是代理方法不同，ISO15693是一种特殊的RFID标签，它拥有自己的协议和数据API(NFCISO15693Tag)。但是该类不起作用。。。可能Apple工程师还在开发中吧

##### Tips
1. 注意同时只能实例化一个读取session(系统会把其他的session放在队列里序列化执行)   
2. Core NFC目前只支持前台扫描，切换到后台会失效    
3. NFCNDEFReaderSession最大每次扫描60s，超时需要重启  
4. 可以配置单一Tag或者多Tag读取模式  
5. 使用提示信息即alertMessage会显示在当前APP的弹出浮层中


获取成功后，即可以根据获取到的信息进行之后的业务流程了。

![CoreNFC读取信息图示](http://ojca2gwha.bkt.clouddn.com/CoreNFC-view.png)



##### 参考
1. https://developer.apple.com/documentation/corenfc
2. https://zh.wikipedia.org/zh/%E8%BF%91%E5%A0%B4%E9%80%9A%E8%A8%8A
3. https://developer.apple.com/videos/play/wwdc2017/718/  
