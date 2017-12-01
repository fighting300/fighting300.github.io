---
title: React Native 原理浅析  
date: 2016-12-01 14:20:00
tags: ReactNative
---

由于Apple严格的审核流程，很多人一直以来都寻找iOS端的动态热发方案，从hybrid到lua、jspatch再到现在非常火的ReactNative，但是动态热发的核心流程都是`从服务端获取配置-->客户端解析-->运行`。目前很多业务的实现例如可配置(config)方案实现，其基本流程都符合以上描述，但是各种方案都有其优劣势，比如可配制方案导致业务逻辑复杂，hybrid体验不太好，jspatch被禁用等。而ReactNative则没有以上这些问题，除了学习成本，你可以很方便快捷的使用js来实现客户端动态热发的功能。

我们团队内部使用ReactNative开发已经一段时间了，开始的时候大概了解过其基础运行原理，但没有深入研究，在工作过程中抽空阅读了源码，从代码层级了解了ReactNative的运行流程，下面分享下其运行的完整流程(依赖于RN：0.41.0版本)。

#### 1.ReactNative原理概述
React Native项目依赖React，React是一个支持JSX语法的用于构建用户界面的JavaScript库，可以认为其相当于MVC框架中的V(视图)。React独创了VirtualDom机制，使他有可能和原生语言互相结合。
而ReactNative让React拥有了于原生APP交互的功能，有了它就能让JS和移动端技术(oc或android)互相调用，这样就可以通过JS开发原生APP的功能，同时还可以支持热发上线。

当然所有的动态热发底层都离不开原生语言(OC/Android)本身，在iOS中，ReactNative之所以能够运行起来，最主要依赖的是Objective-C语言与JavaScript的交互。Apple从iOS7开始，提供了JavaScriptCore的框架，提供了JS和OC沟通的可能，不了解JavaScriptCore的同学可以去了解下。例如以下代码中OC可以直接调用JS代码:   

```
//JSContext为JS代码的运行环境  
JSContext *context = [[JSContext alloc] init];  
JSValue *result = [context evaluateScript:@"2+8"];  
int sum = [result toInt32];  
```
在OC端，可以很容易获取JS上下文，但是JS不知道OC有哪些方法可以调用。ReactNative解决该问题的方法是在OC和JS端保存了一份配置表，里面标记了所有的OC暴露的JS的模块和方法，这样无论那一方调用另一方，实际传递的数据只有ModuleId(模块标志ID)、MethodId(方法标志ID)和Arguments(参数)三个元素，当OC接收到三个参数之后，就可以通过runtime唯一确定要调用的函数并调用。  

另外在RN中，在JS调用OC代码时，会注册要回调的block并且把BlockID作为参数发送给OC，OC收到参数是会创建block，用用完OC后就会执行这个刚刚创建的block。并会向block中传入参数和BlockId，然后在block内部调用JS方法，随后JS查找当时注册的block并执行。


#### 2.源码分析
ReactNative的核心代码包括：
* iOS代码React项目的Base文件夹下，包括`RCTRootView,RCTBridge,RCTBatchedBridge,RCTJSCExecutor`
* JS代码NodeModule中react-native/Libraries下的BatchedBridge和ReactNative文件夹，包括`BatchedBridge,MessageQueue,NativeModules,UIManager`等文件
以上代码流程可以分为初始化阶段和方法调用阶段。  

##### a.初始化ReactNative

要使用ReactNative，需要在OC的入口文件需要用如下代码来进行初始化操作：  
```
RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
                                                      moduleName:@"AwesomeProject"
                                               initialProperties:nil
                                                   launchOptions:launchOptions];
```

用户看到的内容都源于该`RootView`，所有的初始化工作都在其内完成。跳到该方法内部，会发现ReactNative实际上先创建了一个`Bridge`对象，它是OC与JS交互的桥梁，而整个初始化最终的目的就是创建该对象，该对象其实是一个外壳。初始化方法的核心是Birdge对象的`setUp`方法，该方法主要的任务则是创建`BatchedBridge`(0.44版本之后用的是`CxxBridge`)。BatchedBridge的作用是批量读取JS对OC的方法调用，其内部持有`JavaScriptExecutor`对象，用来执行JS代码。
创建BatchedBridge的关键是`start`方法，其流程可以分为五个步骤：  
* 读取JavaScript源码  
* 初始化模块信息
* 初始化JS代码执行器，即`RCTJSCExecutor`对象  
* 生成模块列表并写入JS端  
* 执行JS源码  

###### 读取JavaScript源码      

第一步首页把JavaScript代码家在进内存中这一步中，JSX代码已经被转化成原生的JS代码。
```
NSData *data = [self attemptSynchronousLoadOfBundleAtURL:scriptURL
                                        runtimeBCVersion:JSNoBytecodeFileFormatVersion
                                            sourceLength:&sourceLength
                                                   error:&error];
```

###### 初始化模块信息  
初始化模块在`initModulesWithDispatchGroup`中实现，该方法会找到所有需要暴露给JS的类。每个需要暴露的类都会标记一个宏`RCT_EXPORT_MODULE`，宏的具体实现为：
```
#define RCT_EXPORT_MODULE(js_name) \
RCT_EXTERN void RCTRegisterModule(Class); \
+ (NSString *)moduleName { return @#js_name; } \
+ (void)load { RCTRegisterModule(self); }  
void RCTRegisterModule(Class moduleClass){
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    RCTModuleClasses = [NSMutableArray new];
  });
  // Register module
  [RCTModuleClasses addObject:moduleClass];        
}
```
标记了宏的类会在load时候调用RCTRegisterModule注册自己，这样RN就可以通过RCTModuleClasses拿到所有暴露给JS的类。然后循环遍历该数组，生成RCTModuleData对象。
```
for (Class moduleClass in RCTGetModuleClasses()) {
  NSString *moduleName = RCTBridgeModuleNameForClass(moduleClass);
  RCTModuleData *moduleData = [[RCTModuleData alloc] initWithModuleClass:moduleClass
                                                 bridge:self];
  moduleDataByName[moduleName] = moduleData;
  [moduleClassesByID addObject:moduleClass];
  [moduleDataByID addObject:moduleData];
}
```  
RCTModuleData就是模块配置表的主要内容，该对象保存了每个Module的名字、常量已经所有暴露给JS的方法数组。所有暴露的JS方法需要用`RCT_EXPORT_METHOD`宏来标记，该宏会为函数名加上`__rct_export__`的前缀，再通过runtime获取类的函数列表，找出其中带有指定前缀的方法放入函数数组中。  
```
- (NSArray<id<RCTBridgeMethod>> *)methods{
  unsigned int methodCount;
  Method *methods = class_copyMethodList(object_getClass(_moduleClass), &methodCount);
  for (unsigned int i = 0; i < methodCount; i++) {
    Method method = methods[i];
    SEL selector = method_getName(method);
    if ([NSStringFromSelector(selector) hasPrefix:@"__rct_export__"]) {
      IMP imp = method_getImplementation(method);
      NSArray<NSString *> *entries = ((NSArray<NSString *> *(*)(id, SEL))imp)(_moduleClass, selector);
      id<RCTBridgeMethod> moduleMethod =   [[RCTModuleMethod alloc] initWithMethodSignature:entries[1]
                                                JSMethodName:entries[0]
                                                 moduleClass:_moduleClass];
      [_methods addObject:moduleMethod];
    }
  }
  return _methods;
}
```
以上流程使得Bridge持有一个数组，数组中保存了所有模块的RCTModuleData对象，只要给定ModuleID和MethodId就可以唯一确定要调用的方法。

###### 初始化JS代码执行器   

这一步会初始化`RCTJSCExecutor`对象，该对象被bridge持有。RCTJSCExecutor初始化时会实例化了一个JS线程，用来执行JS代码的线程。该线程执行的runRunLoopThread方法其实是启动了线程的runloop，保证了`_javaScriptThread`能够一直运行。  

```
static NSThread *newJavaScriptThread(void){
  NSThread *javaScriptThread = [[NSThread alloc] initWithTarget:[RCTJSCExecutor class]
                                                       selector:@selector(runRunLoopThread)
                                                         object:nil];
  javaScriptThread.name = RCTJSCThreadName;
  ......
  [javaScriptThread start];
  return javaScriptThread;
}
```
这一步还会进行很多JS上下文的准备，创建全局JSContext并添加很多全局回调block，由JS发起调用。需要重点注意的是`nativeRequireModuleConfig`和`nativeFlushQueueImmediate`这两个block，`nativeRequireModuleConfig`在JS注册新模块的时候调用。
```
   context[@"nativeRequireModuleConfig"] = ^NSArray *(NSString *moduleName) {
     NSArray *result = [strongSelf->_bridge configForModuleName:moduleName];
     return RCTNullIfNil(result);
   };
```
`nativeFlushQueueImmediate`该block用于在js中直接调用OC，注意在RN中JS调用OC一般是由OC发起的，JS会把调用信息放到MessageQueue中等待OC来取。当OC调用了JS后，在对应的JS回调中才会实现调用OC的逻辑。
```
context[@"nativeFlushQueueImmediate"] = ^(NSArray<NSArray *> *calls){
     [strongSelf->_bridge handleBuffer:calls batchEnded:NO];
   };
```
但是当消息队列由等待OC处理的逻辑，并且OC超过5ms没有取走的化，JS就会主动调用OC的以上方法，JS的代码在MessageQueue中如下：  
```
const now = new Date().getTime();
if (global.nativeFlushQueueImmediate && (now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS || this._inCall === 0)){
  var queue = this._queue;
  this._queue = [[], [], [], this._callID];
  this._lastFlush = now;
  global.nativeFlushQueueImmediate(queue);
}
```

###### 生成模块列表并写入JS端  

###### 执行JS源码  

##### b.方法执行阶段  

#### 3.优劣
