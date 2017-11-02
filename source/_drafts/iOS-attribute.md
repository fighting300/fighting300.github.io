---
title: iOS中的attribute
date: 2016-06-12 17:09:03
tags: [iOS,attribute]
categories: iOS
---

在学习开源代码的时候，经常会遇到__attribute__的相关使用，当时在使用的时候又会忘记，所以用这篇文章总结下iOS开发中经常使用的attribute。  
`__attribute__`其实是GNU C的一种机制，是一个编译器的指令，在声明的时候可以提供一些属性，来做多样化的错误检查和高级优化。可以分别用于设置函数属性、变量属性和类型属性。语法一般为: `__attribute__(xx)`。   

1. 函数属性
函数属性可以帮助开发者把一些特性添加到函数声明中，从而可以使编译器在错误检查方面的功能更强大。
**format**
语法为`__attribute__((format(__NSString__, F, A)))`，可以给被声明的函数加上类似printf或者scanf的特征，它可以使编译器检查函数声明和函数实际调用参数之间的格式化字符串是否匹配。`format (archetype, m, n)`，第一个参数传递archetype指定为哪种类型，string-index指定格式化字符串的位置，n指定可变参数检查开始的位置。
例如:
```
#define NS_FORMAT_FUNCTION(F,A) __attribute__((format(__NSString__, F, A)))  
FOUNDATION_EXPORT void NSLog(NSString *format, ...) NS_FORMAT_FUNCTION(1,2);
extern void myPrint(const char *format,...)__attribute__((format(printf,1,2)));
```

**objc_designated_initializer**    
指定内部实现的初始化方法，系统宏`NS_DESIGNATED_INITIALIZER`展开即为该指令，语法为`__attribute__((NS_DESIGNATED_INITIALIZER))`。例如：  
```
- (instancetype)initNoDesignated ;
- (instancetype)initNoDesignated1 NS_DESIGNATED_INITIALIZER;
```
当一个类存在方法带有NS_DESIGNATED_INITIALIZER属性时，它的NS_DESIGNATED_INITIALIZER方法必须调用super的NS_DESIGNATED_INITIALIZER方法。它的其他方法（非NS_DESIGNATED_INITIALIZER）只能调用self的方法初始化。

**objc_requires_super**      
表示子类重写当前类的方法时，必须要调用super函数，否则会有警告。语法为`__attribute__((objc_requires_super))`，例如：
`- (void)fatherMethod __attribute__((objc_requires_super));`

**constructor/destructor**  
表明该函数在mian函数被调用前/后调用，语法为`__attribute__((constructor/destructor))`。  

**warn_unused_result**  
指明一定要使用函数的返回值，否则会有警告，语法为`__attribute__((warn_unused_result))`。例如：  
```
- (int)mustUseReturnValue __attribute__((warn_unused_result)) {
    return 0;
}
```

**nonnull**    
编译器对参数惊醒NULL检查，参数类型必须是指针类型，语法为`__attribute__((nonnull (x)))`，x指第x参数不能为空。例如：  
```
- (int)addNum1:(int *)num1 num2:(int *)num2  __attribute__((nonnull (1))){
    return  *num1 + *num2;
}
```
类似于iOS系统宏NS_ASSUME_NONNULL(参考最后)。  

**sentinel**    
指明变参函数需要nil作为边界，语法为`__attribute__((sentinel))`，例如：  
```
- (void)endWithNil:(nonnull id)first, ... __attribute__((sentinel));
```

**noreturn**
通知编译器当前函数允许不返回值。当遇到类似函数还未运行到return语句就需要退出来的情况，该属性可以避免出现错误信息。语法为`__attribute__((noreturn))`。  
例如：
```
//AFURLConnectionOperation.m
+ (void) __attribute__((noreturn)) networkRequestThreadEntryPoint:(id)__unused object {
    do {
        @autoreleasepool {
            [[NSRunLoop currentRunLoop] run];
        }
    } while (YES);
}
```

**always_inline**   
语法为`__attribute__((always_inline))`。    

**unavailable**      
指明该方法不可用，系统宏`NS_UNAVAILABLE`的展开即为该指令，语法为`__attribute__((unavailable))`。例如：  
```
#define NS_AUTOMATED_REFCOUNT_UNAVAILABLE __attribute__((unavailable("not available in automatic reference counting mode")))  
- (instancetype)init NS_UNAVAILABLE;
- (void)method11 __attribute__((unavailable("不可使用")));
```  

**deprecated**     
编译时会报过时警告，系统宏`DEPRECATED_ATTRIBUTE`的展开即为该指令，语法为`__attribute__((deprecated))`，deprecated后可跟注释。例如：   
```
- (void)method1:( NSString *)string __attribute__((deprecated("使用#method2")));
- (void)method2 DEPRECATED_ATTRIBUTE; //DEPRECATED_ATTRIBUTE是系统的宏
```

2.类型/变量属性

**objc_subclassing_restricted**
指明当前类型不能有子类，相当于final关键字，语法为`__attribute__((objc_subclassing_restricted))`。例如：  
```
__attribute__((objc_subclassing_restricted))
@interface FinalObject : NSObject
```  

**objc_root_class**  
表示当前类是一个根类，例如NSObject，NSProxy等。例如：     
```
//NSObject
__OSX_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0)
OBJC_ROOT_CLASS
@interface NSObject <NSObject> {
    Class isa  OBJC_ISA_AVAILABILITY;
}
```   

**NSObject**  
CFDictionaryRef属于CoreFoundation框架的,也就是非OC对象,加上attribute((NSObject))后,myDictionary的内存管理会被当做OC对象来对待.
例如：`@property (nonatomic,strong) __attribute__((NSObject)) CFDictionaryRef myDictionary;`

**aligned**  
表示所作用的结构陈冠对齐在n字节自然边界上，语法为`__attribute__((aligned(x)))`。例如：  
```
typedef struct
{
    char  member1;
    int   member2;
    short member3;
}__attribute__ ((aligned (8))) Family;
```  

**packed**
指定的结构结构体按照一字节对齐，语法为`__attribute__((packed(x)))`。例如：    
```
typedef struct {
    char    version;
    int16_t sid;
    int32_t len;
    int64_t time;
}__attribute__ ((packed)) Header;
```
常用语与协议相关的网络传输中。  


**cleanup**

作用于变量，当变量的作用域结束是，会调用指定的函数，语法为`__attribute__((cleanup))`。该属性可以作用于block，例如：   
```
- (void)createXcode{
    NSObject *xcode __attribute__((cleanup(xcodeCleanUp))) = [NSObject new];
    NSLog(@"%@",xcode);
}

// 指定一个cleanup方法，注意入参是所修饰变量的地址，类型要一样
// 对于指向objc对象的指针(id *)，如果不强制声明__strong默认是__autoreleasing，会造成类型不匹配  
static void xcodeCleanUp(__strong NSObject **xcode){
    NSLog(@"cleanUp call %@",*xcode);
}
```
相关的示例，可以看下[黑魔法attribute((cleanup))](http://blog.sunnyxx.com/2014/09/15/objc-attribute-cleanup/)






3. Clang Attributes        
Clang Attributes是Clang提供的一种源码注解，方便开发者向编译器表达某种要求，

**availability**  
指明API(变量/方法)版本的变更，语法为`__attribute__((availability(macosx,introduced=m,deprecated=n)))`，m为引入的版本，n为废弃版本，在Cocoa的头文件中可以经常见到。例如：  
`#define CF_DEPRECATED_IOS(_iosIntro, _iosDep, ...) __attribute__((availability(ios,unavailable,introduced=_iosIntro,deprecated=_iosDep,message="" __VA_ARGS__)))`  
其他的参数可以为: obsoleted为移除的版本，unavailable为不可用，message为警告信息。

**overloadable**    
在C的函数上实现像java一样方法重载，语法为`__attribute__((overloadable))`，例如：  
```
__attribute__((overloadable)) void add(int num){
    NSLog(@"Add Int %i",num);
}

__attribute__((overloadable)) void add(NSString * num){
    NSLog(@"Add NSString %@",num);
}

__attribute__((overloadable)) void add(NSNumber * num){
    NSLog(@"Add NSNumber %@",num);
}
```

4. 宏  
**NS_ASSUME_NONNULL**
iOS的系统宏`NS_ASSUME_NONNULL_BEGIN`,`NS_ASSUME_NONNULL_END`，用该宏把整个类包起来，表明当前类的所有指针类型不能显示的置为nil。

```
NS_ASSUME_NONNULL_BEGIN
@interface Food : NSObject
@property (nonatomic, copy) NSString *name;
+ (instancetype)FoodWithName:(NSString *)name;
@end
NS_ASSUME_NONNULL_END
```
通常把这个宏和nullable一起使用。
