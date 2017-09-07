---
title: iOS混淆--OLLVM在iOS中的实践
date: 2017-09-07 12:15:18
tags: -安全
comments: true
---


### OLLVM简介

[OLLVM（Obfuscator-LLVM）](https://github.com/obfuscator-llvm/obfuscator)是瑞士西北应用科技大学安全实验室于2010年6月份发起的一个项目，该项目旨在提供一套开源的针对LLVM的代码混淆工具，以增加对逆向工程的难度。后期转向商业项目[strong.protect](http://link.zhihu.com/?target=https%3A//strong.codes/)。目前，OLLVM已经支持LLVM-4.0版本。

LLVM是一个优秀的编译器框架，它也采用经典的三段式设计。前端可以使用不同的编译工具对代码文件做词法分析以形成抽象语法树AST，然后将分析好的代码转换成LLVM的中间表示IR（intermediate representation）；中间部分的优化器只对中间表示IR操作，通过一系列的Pass对IR做优化；后端负责将优化好的IR解释成对应平台的机器码。LLVM的优点在于，中间表示IR代码编写良好，而且不同的前端语言最终都转换成同一种的IR。

![](http://upload-images.jianshu.io/upload_images/1944396-3ce371d488ad67ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

LLVM IR 是LLVM的中间表示，优化器就是对IR进行操作的，具体的优化操作由一些列的Pass来完成，当前端生成初级IR后，Pass会依次对IR进行处理，最终生成后端可用的IR。下图可以说明这个过程：
![](http://upload-images.jianshu.io/upload_images/1944396-20cd55c8ee11762d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

OLLVM的混淆操作就是在中间表示IR层，通过编写Pass来混淆IR，然后后端依据IR来生成的目标代码也就被混淆了。得益于LLVM的设计，OLLVM适用LLVM支持的所有语言（C,C++,Objective-C,Ada,Fortran）和目标平台（x86,x86-64,PowerPC,PowerPC-64, ARM, Thumb, SPARC, Alpha, CellSPU, MIPS, MSP430, SystemZ, 和 XCore）

#### OLLVM iOS编译环境搭建
以下，介绍OLLVM iOS环境的插件创建过程。
首先下载源码，编译OLLVM混淆器，这里采用LLVM的版本是4.0。下载编译过程如下：

 ```
 $ git clone -b llvm-4.0 https://github.com/obfuscator-llvm/obfuscator.git  
 $ mkdir build  
 $ cd build  
 $ cmake -DCMAKE_BUILD_TYPE=Release ../obfuscator/  
 $ make -j7  
 ```

下载的源码里已经包含了LLVM和Clang，编译完成后，编译好的二进制程序都存在在build/bin目录下。

依据github上的wiki，bin目录下编译好的工具链可以直接用来编译混淆linux下的程序，就像我们常用的gcc那样。若想使用OLLVM来混淆iOS程序，还需将bin目录下的工具链整合进Xcode插件中，具体步骤如下。

配置Xcode--新建Obfuscator插件

```
$ cd /Applications/Xcode.app/Contents/PlugIns/Xcode3Core.ideplugin/Contents/SharedSupport/Developer/Library/Xcode/Plug-ins/  
$ sudo cp -r Clang\ LLVM\ 1.0.xcplugin/ Obfuscator.xcplugin  
$ cd Obfuscator.xcplugin/Contents/  
$ sudo plutil -convert xml1 Info.plist  
$ sudo vim Info.plist  
```
修改Info.plist中的对应内容：

```
<string>com.apple.compilers.clang</string> -> <string>com.apple.compilers.obfuscator</string>  
<string>Clang LLVM 1.0 Compiler Xcode Plug-in</string> -> <string>Obfuscator Xcode Plug-in</string>  
```

修改Obfuscator.xcspec文件

```
$ sudo plutil -convert binary1 Info.plist  
$ cd Resources/  
$ sudo mv Clang\ LLVM\ 1.0.xcspec Obfuscator.xcspec  
$ sudo vim Obfuscator.xcspec  
```
修改ExecPath的地址为当前build/bin的地址(!重点)

```
Identifier = "com.apple.compilers.llvm.clang.1_0"; -> Identifier = "com.apple.compilers.llvm.obfuscator.4_0";  
Name = "Apple LLVM 8.0"; -> Name = "Obfuscator 4.0";  
Description = "Apple LLVM 8.0 compiler"; -> Description = "Obfuscator 4.0";  
Vendor = Apple; -> Vendor = HEIG-VD;  
Version = "7.0"; -> Version = "4.0";  
ExecPath = "clang"; -> ExecPath = "/path/to/obfuscator_bin/clang";  
```

修改 Obfuscator 4.0 Strings

```
$ cd English.lproj/  
$ sudo mv Apple\ LLVM\ 8.0.strings "Obfuscator 4.0.strings"  
$ sudo plutil -convert xml1 Obfuscator\ 4.0.strings  
$ sudo vim Obfuscator\ 4.0.strings  
```

```
"Description" = "Apple LLVM 8.0 Compiler"; -> "Description" = "Obfuscator 4.0";  
"Name" = "Apple LLVM 8.0"; -> "Name" = "Obfuscator 4.0";  
"Vendor" = "Apple"; -> "Vendor" = "HEIG-VD";  
"Version" = "8.0"; -> "Version" = "4.0";  
```
`sudo plutil -convert binary1 Obfuscator\ 4.0.strings`  


现在，你可以打开Xcode在项目配置里来选择新的编译器，并且可以在C Flags或C++ Flags下添加混淆标签。

![选择编译器](http://upload-images.jianshu.io/upload_images/1944396-a27167d3a47b3bc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![添加混淆标签](http://upload-images.jianshu.io/upload_images/1944396-76e1b58511f4a559.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样配置完成后，就可以编译项目生成混淆后的程序。

#### ollvm混淆使用

接下来使用以下demo介绍混淆功能的具体使用。  
```
int main(){  
    int a = 1;  
    int b = 0;  
    int c = 0;  
    if(a > b){  
        a = 100;    
        b = 50;    
        c = a - b;    
        int d = a + b;    
        int e = a & b;    
        int f = a ^ b;    
        printf("c = %d\n",c);    
        printf("d = %d\n",d);    
        printf("e = %d\n",e);    
        printf("f = %d\n",f);    
        printf("a > b\n");    
    }else{  
        printf("a < b\n");    
    }  
    return 0;  
}
```

OLLVM默认支持-fla -sub -bcf 三个混淆参数，这三个参数可以单独使用，也可以配合着使用。-fla 参数表示使用控制流平展（Control Flow Flattening）模式， -sub 参数表示使用指令替换（Instructions Substitution）模式， -bcf 参数表示使用控制流伪造（Bogus Control Flow）模式。

##### Instructions Substitution

指令替换模式主要是将正常的运算操作（+，-，&，|等）替换成功能相等但表述更复杂的形式。比如，对于表达式 a = b + c，它的等价式可以有 a = – ( -b – c), a = b – (-c) 或 a = -(-b) + c 等，原表达式可以替换成任意相等式，或者通过随机数在多个相等式中做选择。SUB模式目前只支持整数运算操作，支持 + , – , & , | 和 ^ 操作，还是比较局限的。编译时，使用 -mllvm -sub 参数即可。

```
-mllvm -sub: 启用instructions substitution  
-mllvm -sub_loop=3: 对每个函数混淆3次，默认1词  
```
![](http://upload-images.jianshu.io/upload_images/1944396-d3ef62eb9e13ae93.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/1944396-a8aff8dc34d9a432.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### Control Flow Flattening

控制流平展模式可以完全改变程序原本的控制流图。如下示例代码是简单的if-else分支语句，正常编译后其控制流图在IDA中下图所示，是正常的if-else分支，使用 -mllvm -fla参数混淆后，在IDA中显示的控制流图如下：
![](http://upload-images.jianshu.io/upload_images/1944396-4a84eba4712effb8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/1944396-47b61ea46b1eee67.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

经FLA模式混淆后，程序的执行流程已经被打乱，出现许多代码分支。通过仔细对比程序混淆前后，可以发现上图着色区域是相对应的，也就是说，FLA模式只去更改代码分支，而不会对单个代码块做处理。

```
-mllvm -fla: 启用control flow flattening
-mllvm -split: 启用block切分，提升平展程度
-mllvm -split_num=3: 对每个block混淆3次，默认1词
```

##### Bogus Control Flow

控制流伪造模式也是对程序的控制流做操作，不同的是，BCF模式会在原代码块的前后随机插入新的代码块，新插入的代码块不是确定的，然后新代码块再通过条件判断跳转到原代码块中。更要命地是，原代码块可能会被克隆并插入随机的垃圾指令。这么多不确定性，就导致对同一份代码多次做BCF模式的混淆时，得到的是不同的混淆效果。可见，BCF混淆模式还是很强大的，不同于FLA那种较确定的混淆模式。使用BCF模式编译时配置参数 -mllvm -bcf即可，此外，BCF模式还支持其它几个参数，下面参数与-mllvm -bcf参数配合使用。  
```
-mllvm -bcf: 启用 bogus control flow
-mllvm -bcf_loop=3: 对一个函数混淆3次，默认1次
-mllvm -bcf_prob=40: 代码块被混淆的概率是40%，默认30%
```
如上图，下面两个着色的代码块就是有上面两个代码块克隆而来，而且其中被插入了一些垃圾指令，类似于这样：


当然，上述介绍的三种混淆模式可以搭配使用，同时使用三个参数混淆后，原本简单的if-else分支代码将会变得异常复杂，这无疑给逆向分析增加巨大的难度。

##### Functions annotations

有的时候，由于效率或其他原因的考虑，我们只想给指定的函数混淆或不混淆该函数，OLLVM也提供了对这一特性的支持，你只需要给对应的函数添加attributes即可。比如，想对函数foo()使用fla混淆，只需要给函数foo()增加fla属性即可。  

```
int foo() __attribute((__annotate__(("fla"))));
int foo() {
   return 2;
}
```

你可以给函数添加一个或多个注释。如果你不想混淆某个函数，你可以使用否定标签。例如如果不想对func()函数使用bcf属性，那标记为“nobcf”即可。



#### 错误处理
1.编译时报错，提示信息如下：  

```
clang-3.6: error: unknown argument: '-gmodules'
clang-3.6: error: unknown argument: '-fembed-bitcode-marker'
Command /Users/dream/ollvm/build/bin/clang failed with exit code 1
```
在Build Settings中搜索并修改：
-gmodules: Obfuscator 4.0 - Code Generation: Generate Debug Symbols: 原来yes，改成no
-fembed-bitcode-marker: Build Option: Enable Bitcode: 原来yes，改成no


你可以在该[git地址]()下找到最新的插件和build编译器文件，该编译器所使用的Xcode版本是8.3.3。
