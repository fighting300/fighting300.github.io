---
title: iOS逆向工程--常用工具  
date: 2017-10-11 10:41:59
tags:
---

这段时间，由于项目中需要加固项目，正好用了一些iOS逆向工程的工具，有些工具不常使用，下面简单总结一下这些工具和使用方法。



#### 1. 查看文件工具  
常见的查看文件工具包括PP助手，iFunbox，iTool，iExplorer等，它们的使用方式都很简单，只需要安装相应的应用，并连接iPhone设备，即可查看当前手机所安装的APP的沙盒(sandbox)文件列表。但是iOS8.3开始，如果你所安装的APP没有开始`iTunes File Shring`选项，相应的app也是无法查看文件内容的，除非在越狱手机上。

![iOS文件列表]()

#### 2. 查看头文件
class-dump  otool   mach-o-view

![iOS文件列表]()

#### 3. 反编译
Hopper Disassembler, IDA Pro

#### 4. UI分析工具 Reveal


#### 5. 网络分析工具
Charles


在正常逆向工程的流程中，要分析一个APP，首先应该砸壳




以后有时间再详细描述下其他逆向工具的使用。


#### 参考文档
1.
