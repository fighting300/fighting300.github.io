---
title: iOS-Bonjour
date: 2017-07-21 18:34:41
tags: iOS
---

之前一直考虑在local现场怎么与别的用户通信，后来陆续了解了苹果的[Bonjour](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/NetServices/Introduction.html)。现在简单写一篇Bonjour的入门介绍。


#### Bonjour架构  

bonjour其实来自法语，是你好的意思。而Bonjour服务是苹果公司发布的一个基于ZEROCONF工作组(IETF下属小组)的工作,用于实现零配置网络联网的解决方案。Bonjour是基于IP层协议的,简单点说,就是某个组织发明了一套解决方案,这套协议能够不需要复杂的配置,即可互相发现彼此的解决方案。可以用它来轻松探测并连接到相同网络中的其他设备，并与别的智能硬件进行交互或者其他操作。典型的Bonjour应用有Remote、AirPrint等。

![]()

为了实现简单配置网络，Bonjour做了以下三点：   
1.寻址(分配IP地址给主机）
一个在网络中的设备需要有一个自己的IP。有了IP地址,我们才能基于IP协议进行通信。对于IPV6标准,IP协议已经包括了自动寻找IP地址的功能。但是目前仍然普遍使用的IPV4不包含本地链路寻址功能。而Bonjour会在本地网络选择一个随机的IP地址进行测试,如果已经被占用,则继续挑选另外一个地址，直到选到可用的IP地址。

2.命名(使用名字而不是IP地址来代表主机）
Bonjour实现了命名和解析功能，保证了我们服务的名字在本地网络是唯一的,并且把别人对我们名字的查询指向正确的IP地址和端口，而不是已IP地址这样不易读的方式来作为服务的标志。
而且Bonjour在系统级别上添加了一个mDNSResponder服务来处理请求和发送回复,从系统级层面上处理,我们就无需在应用内自己监听和返回IP地址,只需到系统内注册服务即可。减少了我们应用的工作量和提高了稳定性。

3.服务搜索（自动在网络搜索服务）
Bonjour可以只需指定所需服务的类型，即可收到本地网络上可用的设备列表。设备在本地网络发出请求,说我需要"XXX"类型的服务,例如：我要打印机服务。所有打印机服务的设备回应自己的名字。

#### 建立连接

简单来说，建立Bonjour连接一般需要三个步骤，即服务端发布服务、客户端浏览服务、客户端/服务端交互。

##### 客户端发布服务

```
    func setupService(){
        let service = NetService.init(domain: "local.", type: "_dragon._tcp", name: "dragon", port: 2333)
        service.delegate = self
        let dictData = "http://fighting300.github.io".data(using: String.Encoding.utf8)
        let data = NetService.data(fromTXTRecord: ["node":dictData!])
        service.setTXTRecord(data)
        service.publish()
        self.service = service
    }
```

```
  func netServiceWillPublish(_ sender: NetService) {
      print("----netServiceWillPublish")
  }
  func netService(_ sender: NetService, didNotPublish errorDict: [String : NSNumber]) {
      print("----netService didNotPublish")
  }


```

##### 客户端发布服务

```
    func netServiceWillPublish(_ sender: NetService) {
        print("----netServiceWillPublish")
    }

    func netServiceDidPublish(_ sender: NetService) {
        print("----netService didPublish")
        let browser = NetServiceBrowser()
        browser.delegate = self
        browser.searchForServices(ofType: "_WE._tcp", inDomain: "local.")
        browser.schedule(in: RunLoop.current, forMode: .commonModes)
        RunLoop.current.run(until: Date.init(timeIntervalSinceNow: 300))
    }

```

```
    func netServiceBrowser(_ browser: NetServiceBrowser, didFind service: NetService, moreComing: Bool) {
            print("----netServiceBrowser didFind", service.domain, service.type, service.name, service.port,service)
            self.service = service
            service.delegate = self
            service.resolve(withTimeout: 5)
        }

        func netServiceBrowser(_ browser: NetServiceBrowser, didRemove service: NetService, moreComing: Bool) {
            print("----netServiceBrowser didRemove")
        }

        func netServiceWillResolve(_ sender: NetService) {
        print("----netService willResolve")
    }

    func netServiceDidResolveAddress(_ sender: NetService) {
        print("----netService didResolveAddress", sender.name, sender.addresses, sender.hostName, sender.addresses?.first)

    }

    func netService(_ sender: NetService, didNotResolve errorDict: [String : NSNumber]) {
        print("----netService didNotResolve ", errorDict)
    }

    func netServiceDidStop(_ sender: NetService) {
        print("----netServiceDidStop")
    }

    func netService(_ sender: NetService, didUpdateTXTRecord data: Data) {
        print("----netService didUpdateTXTRecord")
    }

    func netService(_ sender: NetService, didAcceptConnectionWith inputStream: InputStream, outputStream: OutputStream) {
        print("----netService didAcceptConnectionWith")
    }

    // MARK: util
    func IPFrom(data: Data) -> String {

        return ""
    }

```
##### 客户端/服务端交互

```
```

##### 参考文档
1. http://www.cocoachina.com/ios/20150918/13434.html
2. https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/NetServices/Introduction.html#//apple_ref/doc/uid/TP40002445-SW1
