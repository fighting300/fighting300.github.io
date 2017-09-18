---
title: iOS混淆-ollvm中添加对String的混淆
date: 2017-09-18 15:58:09
tags: 安全
---

之前[研究ollvm](http://fighting300.github.io/2017/09/07/ollvm-in-iOS/)的时候，发现开源的ollvm库中没有对字符串混淆的部分，但是很多APP中都可能会有一些需要加密的字符串。机缘巧合发现上海交大的GoSSIP小组开源了他们设计的基于LLVM4.0的[混淆框架](https://github.com/GoSSIP-SJTU/Armariris),功能包含常量字符串混淆以及ollvm原有的一些功能。[该页面](https://zhuanlan.zhihu.com/p/27617441)有关于他们项目的简介👍。      
简单分析发现字符串混淆的部分主要由字符串加密的Pass完成，之后我们考虑把字符串混淆的功能加入ollvm中，所以本文简单介绍下如何将字符串加密的Pass继承到ollvm中。Pass其实可以简单的理解为LLVM优化／转换工作的一个最小单元，可以把所有的混淆工作都是由一个一个Pass组成的，想要做具体的了解，可以看下以下[专栏的文章](http://www.nagain.com/activity/article/14/)。

#### Pass集成

首先提取字符串加密文件对应pass文件，该文件所在目录为lib/Transform/Obfuscation。
![pass文件目录](http://ojca2gwha.bkt.clouddn.com/ollvm-string-cpp.png)

<!--more-->
提取后放在ollvm相同目录下，并把头文件也复制到对应目录下，该文件所在目录为include/llvm/Transform/Obfuscation。  
![头文件目录](http://ojca2gwha.bkt.clouddn.com/ollvm-string-h.png)

修改lib/Transform/Obfuscation目录下的CMakeLists.txt文件，将StringObfuscation.cpp添加到编译库中。然后修改Transform/IPO下的PassManagerBuilder.cpp文件，添加字符串加密的编译代码。具体代码如下:

##### 1.添加引用
`#include "llvm/Transforms/Obfuscation/StringObfuscation.h"`

##### 2.插入函数声明，即编译时的编译参数`-mllvm -sobf`

```
static cl::opt<std::string> Seed("seed", cl::init(""),
                           cl::desc("seed for the random"));

static cl::opt<bool> StringObf("sobf", cl::init(false),
                           cl::desc("Enable the string obfuscation"));
```

##### 3.在PassManagerBuilder()构造函数中添加随机数因子的初始化  

```
    if(!Seed.empty()) {
      llvm::cryptoutils->prng_seed(Seed.c_str());
    }
```

![插入随机因子初始化](http://ojca2gwha.bkt.clouddn.com/ollvm-string-seed.png)

##### 4.添加pss到 PassManagerBuilder::populateModulePassManager中

```
    MPM.add(createStringObfuscation(StringObf));
```

![添加Pass](http://ojca2gwha.bkt.clouddn.com/ollvm-string-use.png)

##### 5.在主路径下运行编译命令  

```
  mkdir build
  cd build
  cmake -DCMAKE_BUILD_TYPE=Release ../obfuscator/   // 代码所在路径
  make -j7
```
即可生成build最终文件，用于项目编译。我在github上也新建了一个添加了字符串混淆功能的[ollvm版本](https://github.com/fighting300/obfuscator)，想要直接使用的小伙伴可以下载使用。

##### 6.使用效果  
在OC代码中添加了简单的字符串、混淆后的效果如下图：  

![字符串混淆前](http://ojca2gwha.bkt.clouddn.com/ollvm-string-before.png)

![字符串混淆后](http://ojca2gwha.bkt.clouddn.com/ollvm-string-after.png)


#### 问题及解决方案  

编译成功后，则可按照[上篇文章](http://fighting300.github.io/2017/09/07/ollvm-in-iOS/)所述，在XCode中编译项目。   

##### 1. bcf不支持Invoke指令
在实际使用过程中，发现ollvm目前不支持@synchronized、try...catch等少数语法，然后导致bcf报错。[这些语法](https://llvm.org/docs/LangRef.html#invoke-instruction)会生成invoke指令，目前可以在bcf前过滤包含InvokeInst的方法，具体代码可以参考[该Github地址](https://github.com/fighting300/obfuscator/commit/ae0e5acd873cd9a8c839a013a635422022fd0d6b)。

##### 2.目前该字符串识别还有bug，即不会加密每个函数的第一个字符串。。。[该bug](https://qtfreet.com/?p=313)未经过验证。

若有其他问题，可以联系，或者comment(翻墙才能评论哈。。。)

##### 参考文档
1. http://bobao.360.cn/learning/detail/4069.html
2. http://www.nagain.com/activity/article/14/
