---
title: iOS Bonjour的使用-本地通信/智能交互
date: 2017-07-21 18:34:41
tags: iOS
---

之前一直考虑在local现场怎么与别的用户通信，后来陆续了解了苹果的[Bonjour](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/NetServices/Introduction.html)。现在简单写一篇Bonjour的入门介绍。

#### Bonjour介绍  

bonjour其实来自法语，是你好的意思。而Bonjour服务是苹果公司发布的一个基于ZEROCONF工作组(IETF下属小组)的工作,用于实现零配置网络联网的解决方案。Bonjour是基于IP层协议的,简单来说,就是一套解决方案,能够不需要复杂的配置,即可互相发现彼此的解决方案。可以用它来轻松探测并连接到相同网络中的其他设备，并与别的智能硬件进行交互或者其他操作。典型的Bonjour应用有Remote、AirPrint等。

![Bonjour-overview](http://ojca2gwha.bkt.clouddn.com/iOS-bonjour-Overview.png)

为了实现简单配置网络，Bonjour做了以下三点：   
1. 寻址(分配IP地址给主机）
一个在网络中的设备需要有一个自己的IP。有了IP地址,我们才能基于IP协议进行通信。对于IPV6标准,IP协议已经包括了自动寻找IP地址的功能。但是目前仍然普遍使用的IPV4不包含本地链路寻址功能。而Bonjour会在本地网络选择一个随机的IP地址进行测试,如果已经被占用,则继续挑选另外一个地址，直到选到可用的IP地址。

2. 命名(使用名字而不是IP地址来代表主机）
Bonjour还实现了命名和解析功能，保证了我们服务的名字在本地网络是唯一的,并且把别人对我们名字的查询指向正确的IP地址和端口，而不是以IP地址这样不易读的方式来作为服务的标志。
而且Bonjour在系统级别上添加了一个mDNSResponder服务来处理请求和发送回复,从系统级层面上处理,我们就无需在应用内自己监听和返回IP地址,只需到系统内注册服务即可。减少了我们应用的工作量和提高了稳定性。

3. 服务搜索（自动在网络搜索服务）
Bonjour可以只需指定所需服务的类型，即可收到本地网络上可用的设备列表。设备在本地网络发出请求,说我需要"XXX"类型的服务,例如：我要打印机服务。所有打印机服务的设备回应自己的名字。

<!--more-->
#### Bonjour API

Cocoa中网络框架有三层，最底层的是基于 BSD socket库，然后是 Cocoa 中基于 C 的 CFNetwork，最上面一层是 Cocoa 中 Bonjour。通常我们无需与 socket 打交道，我们会使用经 Cocoa 封装的 CFNetwork 和 Bonjour 来完成大多数工作。  
> cocoa 很多组件都有两种实现，一种是基于 C 的以 CF 开头的类（CF=Core Foundation），这是比较底层的；另一种是基于 Obj-C 的以 NS 开头的类(NS=Next Step)，这种类抽象层次更高，易于使用。对于网络框架也一样。Bonjour 中 NSNetService 也有对应的 CFNetService，NSInputStream 有对应的 CFInputStream。  

通过 Bonjour，一个应用程序publish一个网络服务 service，然后网络中的其他程序就能自动发现这个 service，从而可以向这个 service 查询其 ip 和 port，然后通过获得的 ip 和 port 建立 socket 链接进行通信。通常我们是通过 NSNetService 和 NSNetServiceBrowser 来使用 Bonjour 的，前者用于建立与发布 service，后者用于监听查询网络上的 service。

![Bonjour-API](http://ojca2gwha.bkt.clouddn.com/iOS-bonjour-API.png)

#### 建立连接

简单来说，建立Bonjour连接一般需要三个步骤，即服务端发布服务、客户端浏览服务、客户端/服务端交互。

![Bonjour-API](http://ojca2gwha.bkt.clouddn.com/iOS-bonjour-API1.png)
##### 客户端发布服务

首先通过`NetService`对象初始化服务，指定服务的域、类型、名称和端口，在同一网络中，服务类型名必须唯一，这样才能精准定位服务。
Bonjour操作也需要异步进行，以免长时间阻碍主线程，所以我们将发布任务交给当前run loop去调度。   

```
    func setupService(){
        let service = NetService.init(domain: "local.", type: "_dragon._tcp", name: "dragon", port: 2333)
        service.schedule(in: RunLoop.current, forMode: .commonModes)
        service.delegate = self
        let dictData = "http://fighting300.github.io".data(using: String.Encoding.utf8)
        let data = NetService.data(fromTXTRecord: ["node":dictData!])
        service.setTXTRecord(data)
        service.publish()
        self.service = service
    }
```

另外需要实现NetService协议`NetServiceDelegate`的代理方法跟踪服务发布信息。  

```
  func netServiceWillPublish(_ sender: NetService) {
      print("----netServiceWillPublish")
  }
  func netService(_ sender: NetService, didNotPublish errorDict: [String : NSNumber]) {
      print("----netService didNotPublish")
  }

```

##### 客户端浏览服务

服务发布成功后，会在代理方法中接受到发布的消息，这时候要在客户端通过`NetServiceBrowser`对象来浏览本地的服务，并展示本地网络中可用的服务。
可以通过`searchForServices`方法指定需要查找的服务类型和查找的域，然后运行在mainRunLoop中。

```
    func netServiceDidPublish(_ sender: NetService) {
        print("----netService didPublish")
        let browser = NetServiceBrowser()
        browser.delegate = self
        browser.searchForServices(ofType: "_WE._tcp", inDomain: "local.")
        browser.schedule(in: RunLoop.current, forMode: .commonModes)
        RunLoop.current.run(until: Date.init(timeIntervalSinceNow: 300))
    }

```

同时实现NetServiceBrowser的代理`NetServiceBrowserDelegate`方法`netServiceBrowser(_ browser: NetServiceBrowser, didFind service: NetService, moreComing: Bool)`来处理相应服务的解析。当前代码实例中没有选择服务的服务，直接对扫描到的服务来做解析。

```
    func netServiceBrowser(_ browser: NetServiceBrowser, didFind service: NetService, moreComing: Bool) {
        print("----netServiceBrowser didFind", service.domain, service.type, service.name, service.port,service)
        // 在界面选择对应的service
        self.service = service
        service.delegate = self
        service.resolve(withTimeout: 5)
    }

    func netServiceBrowser(_ browser: NetServiceBrowser, didRemove service: NetService, moreComing: Bool) {
        print("----netServiceBrowser didRemove")
    }
```
##### 客户端/服务端交互

最后，可以在NetService代理的解析方法里`func netServiceDidResolveAddress(_ sender: NetService)`，拿到名称、类型、域、主机名和ip地址等信息。

```
    func netServiceWillResolve(_ sender: NetService) {
        print("----netService willResolve")
    }

    func netServiceDidResolveAddress(_ sender: NetService) {
        print("----netService didResolveAddress", sender.name, sender.addresses, sender.hostName, sender.addresses?.first)
        let data = sender.txtRecordData()
        let dict = NetService.dictionary(fromTXTRecord: data!)
        let info = String.init(data: dict["node"]!, encoding: String.Encoding.utf8)
        print("mac info = ",info);

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

    // MARK: util 获取ip地址
    func IPFrom(data: Data) -> String {
      let dataIn: NSData = data as NSData
      var storage = sockaddr_storage()
      dataIn.getBytes(&storage, length: MemoryLayout<sockaddr_storage>.size)
      if Int32(storage.ss_family) == AF_INET {
          let addr4 = withUnsafePointer(to: &storage) {
              $0.withMemoryRebound(to: sockaddr_in.self, capacity: 1) {
                  $0.pointee
              }
          }
          let ipString =  String(cString: inet_ntoa(addr4.sin_addr), encoding: .ascii)
          print("ip", ipString)
          return ipString
      }
      return ""
    }
```

之后依靠以上获取的信息，需要通过Socket/Streams建立连接来进行通信，本篇文章不对这部分做更多的介绍，后续有时间再补充完整。

##### 参考    
1. http://mobileorchard.com/tutorial-networking-and-bonjour-on-iphone/
2. https://developer.apple.com/library/content/documentation/Networking/Conceptual/NSNetServiceProgGuide/Introduction.html#//apple_ref/doc/uid/TP40002520-SW2
3. http://www.cocoachina.com/ios/20150918/13434.html
