---
title: React Native源码分析
tags: ReactNative
date: 2017-03-02 14:20:00
---


由于Apple严格的审核流程，很多人一直以来都寻找iOS端的动态热发方案，从Hybrid到Lua、JSPatch再到现在非常火的ReactNative，但是动态热发的核心流程都是`从服务端获取配置-->客户端解析-->运行`。目前很多业务的实现例如可配置方案实现，其基本流程都符合以上流程，但是各种方案都有其优劣势，比如可配制方案导致业务逻辑复杂，Hybrid体验不太好，JSPatch被禁用等。而ReactNative则没有以上这些问题，除了学习成本，你可以很方便快捷的使用JS来实现客户端动态热发的功能。

我们团队内部使用ReactNative开发已经一段时间了，开始的时候大概了解过其基础运行原理，但没有深入研究，在工作过程中抽空阅读了源码，从代码层级了解了ReactNative的运行流程，下面分享下其运行的完整流程(依赖于RN：0.41版本)。

#### ReactNative原理概述
React Native项目依赖React，React是一个支持JSX语法的用于构建用户界面的JS库，可以认为其相当于MVC框架中的View(视图)。React独创了虚拟Dom机制，使他有可能和原生语言互相结合。
而ReactNative让React拥有了于原生APP交互的功能，有了它就能让JS和移动端技术(iOS或Android)互相调用，这样就可以通过JS开发原生APP的功能，同时还可以支持热发上线。React Native也秉承了React的理念，`Learn Once,Write Anywhere`。

当然所有的动态热发底层都离不开原生语言(iOS/Android)本身，在iOS中，ReactNative之所以能够运行起来，最主要依赖的是Objective-C语言与JavaScript的交互。Apple从iOS7开始，提供了JavaScriptCore的框架，提供了JS和OC沟通的可能，不了解JavaScriptCore的同学可以去了解下。例如iOS可以通过以下代码直接运行JS代码:   

```
//JSContext为JS代码的运行环境  
JSContext *context = [[JSContext alloc] init];  
JSValue *result = [context evaluateScript:@"2+8"];  
int sum = [result toInt32];  
```

在OC端，可以很容易获取JS上下文，但是JS不知道OC有哪些方法可以调用。ReactNative解决该问题的方法是在OC和JS端保存了一份配置表，里面标记了所有的OC暴露的JS的模块和方法，这样无论那一方调用另一方，实际传递的数据只有ModuleId(模块标志ID)、MethodId(方法标志ID)和Arguments(参数)三个元素，当OC接收到三个参数之后，就可以通过runtime唯一确定要调用的函数并调用。  

另外在RN中JS调用OC代码时，会注册要回调的block并且把BlockID作为参数发送给OC，OC收到参数是会创建block，用用完OC后就会执行这个刚刚创建的block。并会向block中传入参数和BlockId，然后在block内部调用JS方法，随后JS查找当时注册的block并执行。

#### 源码分析
ReactNative的核心代码包括：  
> OC代码：在React项目的Base文件夹下，包括`RCTRootView`,`RCTBridge`,`RCTBatchedBridge`,`RCTJSCExecutor`(0.44之后RCTBatchedBridge，改为RCTCxxBridge)  
> JS代码：NodeModule中react-native/Libraries下的BatchedBridge和ReactNative文件夹，包括`BatchedBridge`,`MessageQueue`,`NativeModules`,`UIManager`等文件    

以上代码流程可以分为初始化阶段和方法调用阶段。  

##### 初始化ReactNative环境  
要使用ReactNative创建View，首先需要在OC的入口文件需要用如下代码来进行初始化操作：    
```
RCTRootView *rootView = [[RCTRootView alloc] initWithBundleURL:jsCodeLocation
                                                      moduleName:@"AwesomeProject"
                                               initialProperties:nil
                                                   launchOptions:launchOptions];
```

用户看到的内容都源于该`RootView`，所有的初始化工作都在其内完成。进入该方法内部，会发现ReactNative实际上先创建了一个`Bridge`对象，它是OC与JS交互的桥梁，而整个初始化最终的目的就是创建该对象，该对象其实是一个外壳。初始化方法的核心是Birdge对象的`setUp`方法，该方法主要的任务则是创建`BatchedBridge`(0.44版本之后用的是`CxxBridge`)。BatchedBridge的作用是批量读取JS对OC的方法调用，其内部持有`JSCExecutor`对象，用来执行JS代码。
创建BatchedBridge的关键是`start`方法，其流程可以分为五个步骤：  
* (1)读取JS源码  
* (2)初始化模块信息
* (3)初始化JS代码执行器，即`RCTJSCExecutor`对象  
* (4)生成模块列表并写入JS端  
* (5)执行JS源码  

###### 读取JavaScript源码      

第一步首页把JS代码加载进内存中，这一步中，JSX代码已经被转化成原生的JS代码。这一部分很简单，不用多说。
```
[self loadSource:^(NSError *error, RCTSource *source) {
  sourceCode = source.data;
}];
// loadSource内部实际调用方法为：  
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
// RCTRegisterModule具体实现：  
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

RCTModuleData就是模块配置表的主要内容，该对象保存了每个Module的名字、常量已经所有暴露给JS的方法数组。所有暴露的JS方法需要用`RCT_EXPORT_METHOD`宏来标记，该宏会为函数名加上`__rct_export__`的前缀，后续再通过runtime获取类的函数列表，找出其中带有指定前缀的方法放入函数数组中。

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

这一步先创建了一个group，用于执行该步及下一步的操作，(`setUpExecutor`)方法会初始化`RCTJSCExecutor`对象，该对象被bridge持有，所有的JS与OC的通信都是通过该类来实现的。RCTJSCExecutor初始化时会实例化了一个JS线程，用来执行JS代码的线程。该线程执行的runRunLoopThread方法其实是启动了JS线程的runloop，保证了`_javaScriptThread`能够一直运行。  
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

这一步还会进行很多JS上下文的准备，创建全局JSContext并添加很多全局回调block，由JS发起调用。需要重点注意的是`nativeRequireModuleConfig`和`nativeFlushQueueImmediate`这两个block，其中`nativeRequireModuleConfig`在JS注册新模块时调用。  
```
   context[@"nativeRequireModuleConfig"] = ^NSArray *(NSString *moduleName) {
     NSArray *result = [strongSelf->_bridge configForModuleName:moduleName];
     return RCTNullIfNil(result);
   };
```
`nativeFlushQueueImmediate`该block用于在JS中直接调用OC，注意在RN中JS调用OC一般是由OC发起的，JS会把调用信息放到MessageQueue中等待OC来取。当OC调用了JS后，在对应的JS回调中才会实现调用OC的逻辑。
```
context[@"nativeFlushQueueImmediate"] = ^(NSArray<NSArray *> *calls){
     [strongSelf->_bridge handleBuffer:calls batchEnded:NO];
   };
```
但是当消息队列由等待OC处理的逻辑，并且OC超过5ms没有取走的话，JS就会主动调用OC的以上方法，该部分JS的代码在MessageQueue中如下：  
```
const now = new Date().getTime();
if (global.nativeFlushQueueImmediate && (now - this._lastFlush >= MIN_TIME_BETWEEN_FLUSHES_MS || this._inCall === 0)){
  var queue = this._queue;
  this._queue = [[], [], [], this._callID];
  this._lastFlush = now;
  global.nativeFlushQueueImmediate(queue);
}
```
该handleBuffer方法是JS调用OC方法的关键，后续调用模块会提及。注意在这个里面使用到的JSC_JSXXX的宏的作用，实际上是会转换为调用苹果的JavaScriptCore对应的方法(去掉JSC_)。

###### 生成模块列表并写入JS端  
这一步就是让JS获取所有模块的名字。首先调用`moduleConfig`方法获取当前module的config数据：  
```
- (NSString *)moduleConfig{
  NSMutableArray<NSArray *> *config = [NSMutableArray new];
  for (RCTModuleData *moduleData in _moduleDataByID) {
    if (self.executorClass == [RCTJSCExecutor class]) {
      [config addObject:@[moduleData.name]];
    } else {
      [config addObject:RCTNullIfNil(moduleData.config)];
    }
  }
  return RCTJSONStringify(@{@"remoteModuleConfig": config,}, NULL);
}
```
然后调用`injectJSONConfiguration`方法把config注入JS执行环境中，保存为`__fbBatchedBridgeConfig`，之后在NativeModule.js中会用到该属性。
```
[_javaScriptExecutor injectJSONText:configJSON
                asGlobalObjectNamed:@"__fbBatchedBridgeConfig"
                           callback:onComplete];
```
###### 执行JS源码   
modules和JS代码准备好后，会在dispatch_group_notify的JS子线程内执行jsBundle代码。执行完毕后会将_displayLink添加到runloop(注意是在jsThread所在的runloop)中，开始运行。运行代码时，第三步中的block会被执行，从而向JS端写入配置信息。这样JS和OC就具备了交互的能力，准备工作到此完成。具体代码如下：
```
[_javaScriptExecutor executeApplicationScript:script sourceURL:url onComplete:^(NSError *scriptLoadError) {

  [self->_javaScriptExecutor flushedQueue:^(id json, NSError *error){
     [self handleBuffer:json batchEnded:YES];
     onComplete(error);
   }];
}];

```

##### 方法执行阶段  
上面的工作完成之后，初始化bridge的工作就完成了，jsBundle已经加载完成，OC端的配置表也已经处理好，并且成功已经传递给JS端，JS的上下文配置已经准备好，下面就开始执行JS代码了。React就会开始计算好所有的布局信息，以及Component层级关系等，等待native端完成对应的真正的页面渲染和布局。回到RCTRootView的初始化方法中，注意在[RCTRootView initWithBridge:...]的初始化方法中，注册了几个JS执行情况的通知，我们重点关注JS执行完毕后的通知RCTJavaScriptDidLoadNotification。

```
- (void)bundleFinishedLoading:(RCTBridge *)bridge{
  _contentView = [[RCTRootContentView alloc] initWithFrame:self.bounds  bridge:bridge
                                                reactTag:self.reactTag  sizeFlexiblity:_sizeFlexibility];
  [self runApplication:bridge];
  [self insertSubview:_contentView atIndex:0];
}

```

在创建RCTRootContentView的时候，注意有个参数是reactTag，这个属性很重要，每一个reactTag都应该是唯一的，从1开始每次递增10。RCTRootContentView初始化时，还需要在RCTUIManager中通过reactTag去注册，从而由RCTUIManager来统一管理所有的JS端使用Component对应的每个原生view(`_viewRegistry[tag]`表)，有了这个我们就可以很方便的在其他地方通过reactTag获取到我们的Component所在的rootView.
```
- (NSNumber *)allocateRootTag{
  NSNumber *rootTag = objc_getAssociatedObject(self, _cmd) ?: @1;
  objc_setAssociatedObject(self, _cmd, @(rootTag.integerValue + 10), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
  return rootTag;
}
```
然后才是[RCTRootView runApplication]，这里就会调用JS里面AppRegistry对应的方法runApplication了。
```
- (void)runApplication:(RCTBridge *)bridge{
  NSString *moduleName = _moduleName ?: @"";
  NSDictionary *appParameters = @{
    @"rootTag": _contentView.reactTag,
    @"initialProps": _appProperties ?: @{},
  };
  RCTLogInfo(@"Running application %@ (%@)", moduleName, appParameters);
  [bridge enqueueJSCall:@"AppRegistry"
                 method:@"runApplication"
                   args:@[moduleName, appParameters]
             completion:NULL];
}
```

执行JS代码的时候，JS会计算好每个View的布局属性等信息，然后通过调用native的系统方法来完成页面渲染布局。这个过程就设置到JS和Native的相互通信了。如前文所述，在RN中，OC和JS的交互都是通过传递ModuleId，MethodId和Arguments进行的。

###### 调用JS代码
在OC中，JS代码一直在一个单独的子线程上面运行，如下
```
- (void)executeBlockOnJavaScriptQueue:(dispatch_block_t)block{
  if ([NSThread currentThread] != _javaScriptThread) {
    [self performSelector:@selector(executeBlockOnJavaScriptQueue:)
                 onThread:_javaScriptThread withObject:block waitUntilDone:NO];
  } else {
    block();
  }
}
```
调用JS代码的核心函数如下：
```
- (void)_executeJSCall:(NSString *)method arguments:(NSArray *)arguments unwrapResult:(BOOL)unwrapResult callback:(RCTJavaScriptCallback)onComplete{
  __weak RCTJSCExecutor *weakSelf = self;
  [self executeBlockOnJavaScriptQueue:^{
    RCTJSCExecutor *strongSelf = weakSelf;
    JSContext *context = strongSelf->_context.context;
    JSGlobalContextRef ctx = context.JSGlobalContextRef;

    JSValueRef errorJSRef = NULL;
    JSValueRef batchedBridgeRef = strongSelf->_batchedBridgeRef;

    JSValueRef methodJSRef = JSC_JSObjectGetProperty(ctx, (JSObjectRef)batchedBridgeRef, methodNameJSStringRef, &errorJSRef);
    JSValueRef jsArgs[arguments.count];
    for (NSUInteger i = 0; i < arguments.count; i++) {
      jsArgs[i] = [JSC_JSValue(ctx) valueWithObject:arguments[i] inContext:context].JSValueRef;
    }
    JSValueRef resultJSRef = JSC_JSObjectCallAsFunction(ctx, (JSObjectRef)methodJSRef, (JSObjectRef)batchedBridgeRef, arguments.count, jsArgs, &errorJSRef);

    JSValue *result = [JSC_JSValue(ctx) valueWithJSValueRef:resultJSRef inContext:context];
    id objcValue = unwrapResult ? [result toObject] : result;

    onComplete(objcValue, error);
  }];
}
```
需要注意的是，这个函数名是我们要调用JS的中转函数名，比如`callFunctionReturnFlushedQueue`。也就是说它的作用其实是处理参数，而非真正要调用的 JS函数。
这个中转函数接收到的参数包含了ModuleId、MethodId和Arguments，然后由中转函数查找自己的模块配置表，找到真正要调用的JS函数。


###### JS调用OC代码
JS代码中`MessageQueue.js`和`NativeModule.js`是核心文件。`BatchedBridge.js`文件仅仅是MessageQueue.js的导入文件，不用关注。  
MessageQueue文件是js端与oc通信的一个桥梁文件，里边有个计时器将要调用oc方法的事件放入一个queue中。
NativeModule文件实现了JS调用OC代码。需要注意的是这个调用其实都是OC先调用JS文件，执行JS，在这时候传入适当的参数，然后实现了JS调用OC方法。
在调用OC代码时，JS会解析出方法的ModuleId、MethodId和Arguments并放入到MessageQueue中，等待OC主动拿走，或者超时后主动发送给OC。

OC负责处理调用的方法是`handleBuffer`，它的参数是一个含有四个元素的数组，每个元素也都是一个数组，分别存放了ModuleId、MethodId、Params，第四个元素目测用处不大。
函数内部在每一次方调用中调用`_handleRequestNumber:moduleID:methodID:params`方法，通过查找模块配置表找出要调用的方法，并通过runtime动态的调用：

```
[method invokeWithBridge:self module:moduleData.instance arguments:params];
```
在这个方法中，有一个很关键的方法：`processMethodSignature`，它会根据JS的CallbackId创建一个Block，并且在调用完函数后执行这个Block。


`RCTUIManager.h, UIManage.js`这两个类一个是在oc端，一个是在js端，通过这两个类RN将我们的component转换成了我们原生的视图类。该类实现了视图的创建、查找、删除等功能。创建View的时候，每个View都有一个

最新的0.48版本的总体流程和以上流程差别不大，主要是替换了初始化部分的执行顺序，大家可以自行参阅最新的代码。


##### 参考文档  
1. [React Native通信机制详解](http://blog.cnbang.net/tech/2698/)   
2. [React Native源码分析](http://www.jianshu.com/p/0688b24950f4)
