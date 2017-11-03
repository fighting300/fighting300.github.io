---
title: iOS中的attribute和宏
tags:
  - iOS
  - attribute
  - 宏
categories: iOS
date: 2016-06-12 17:09:03
---

在学习开源代码的时候，经常会遇到`__attribute__`的相关使用，但是在使用的时候又会忘记，所以用这篇文章总结下iOS开发中经常使用的attribute。  
`__attribute__`其实是GNU C的一种机制，本质是一个编译器的指令，在声明的时候可以提供一些属性，在编译阶段起作用，来做多样化的错误检查和高级优化。可以分别用于设置函数属性、变量属性和类型属性，语法一般为: `__attribute__(xx)`。下面分别介绍下这些属性。   

#### 函数属性  
函数属性可以帮助开发者把一些特性添加到函数声明中，从而可以使编译器在错误检查方面的功能更强大。   

**format**
语法为`__attribute__((format(__NSString__, F, A)))`，可以给被声明的函数加上类似printf或者scanf的特征，它可以使编译器检查函数声明和函数实际调用参数之间的格式化字符串是否匹配。`format (archetype, m, n)`，第一个参数传递archetype指定为哪种类型，string-index指定格式化字符串的位置，n指定可变参数检查开始的位置。例如:  
```
#define NS_FORMAT_FUNCTION(F,A) __attribute__((format(__NSString__, F, A)))  
FOUNDATION_EXPORT void NSLog(NSString *format, ...) NS_FORMAT_FUNCTION(1,2);
extern void myPrint(const char *format,...)__attribute__((format(printf,1,2)));
```

**objc_designated_initializer**    
指定内部实现的初始化方法，系统宏`NS_DESIGNATED_INITIALIZER`展开即为该指令，语法为`__attribute__((objc_designated_initializer))`。例如：  
```
- (instancetype)initNoDesignated ;
- (instancetype)initNoDesignated1 NS_DESIGNATED_INITIALIZER;
```
当一个类存在方法带有NS_DESIGNATED_INITIALIZER属性时，它的NS_DESIGNATED_INITIALIZER方法必须调用super的NS_DESIGNATED_INITIALIZER方法。它的其他方法（非NS_DESIGNATED_INITIALIZER）只能调用self的方法初始化。

**objc_requires_super**      
表示子类重写当前类的方法时，必须要调用super函数，否则会有警告。语法为`__attribute__((objc_requires_super))`，例如：   
`- (void)fatherMethod __attribute__((objc_requires_super));`   

<!--more-->  

**constructor/destructor**  
表明该函数在mian函数被调用前/后调用，语法为`__attribute__((constructor/destructor))`。例如：  
```
static __attribute__((constructor)) void before() {
    printf("Hello");
}
static __attribute__((destructor)) void after() {
    printf(" World!");
}
int main(int argc, const char * argv[]) {
    printf("1");
    return 0;
}
```
constructor和`load`方法都是在main函数执行前调用，但是`load`更早一些。

**warn_unused_result**  
指明一定要使用函数的返回值，否则会有警告，语法为`__attribute__((warn_unused_result))`。例如：  
```
- (int)mustUseReturnValue __attribute__((warn_unused_result)) {
    return 0;
}
```

**nonnull**    
编译器对参数进行NULL检查，参数类型必须是指针类型，语法为`__attribute__((nonnull (x)))`，x指第x参数不能为空。例如：  
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

**always_inline/noinline**   
表示函数强制内联，语法为`__attribute__((always_inline/noinline))`。例如：`void test(int a) __attribute__((always_inline));`。      
编译器编译内联函数时会直接将函数插入到调用的地方，所以没有普通函数调用时的额外开销(压栈/跳转/返回)。但是是否形成内联函数，需要看编译器对该函数定义的具体处理，编译器可能会拒绝内联请求。

**unavailable**      
指明该方法不可用，系统宏`NS_UNAVAILABLE`的展开即为该指令，语法为`__attribute__((unavailable))`。例如：   
```
#define NS_AUTOMATED_REFCOUNT_UNAVAILABLE __attribute__((unavailable("not available in automatic reference counting mode")))  
- (instancetype)init NS_UNAVAILABLE;
- (void)method11 __attribute__((unavailable("不可使用")));
```

**deprecated**     
表示当前方法/属性不推荐使用，编译时会报过时警告，系统宏`DEPRECATED_ATTRIBUTE`、`NS_DEPRECATED`的展开即为该指令，语法为`__attribute__((deprecated))`，deprecated后可跟注释。例如：    
```
- (void)method1 __attribute__((deprecated("使用#method2")));
- (void)method2 DEPRECATED_ATTRIBUTE; //DEPRECATED_ATTRIBUTE是系统的宏
```

**enable_if**  
用来实现C函数参数的静态检查，例如：   
```
static void printValidAge(int age)
__attribute__((enable_if(age > 0, "当前参数有问题～"))) {
    printf("%d", age);
}
```

在函数声明中，可以同时使用多个属性，用`,`隔开即可。

#### 类型/变量属性

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
CFDictionaryRef属于CoreFoundation框架的,也就是非OC对象,加上attribute((NSObject))后,myDictionary的内存管理会被当做OC对象来对待。例如：  
`@property (nonatomic,strong) __attribute__((NSObject)) CFDictionaryRef myDictionary;`

**objc_runtime_name**  
可以修改类/协议在runtime中的名称，语法为`__attribute__((objc_runtime_name("xxx")))`，可以使用该特性做代码混淆。例如：   
```
__attribute__((objc_runtime_name("NewFather")))
@interface Father : NSObject
@end
```

**unused**  
表示变量即使不使用，也不会产生警告，语法为`__attribute__((unused))`。例如：`NSInteger i __attribute__((unused)) = 10;`。   

**objc_boxable**
可以将struct/union类型包装成NSValue对象，语法为`__attribute__((objc_boxable))`。例如：  
```
typedef struct __attribute__((objc_boxable)) {
    CGFloat x, y, width, height;
} XXRect;
```
这样就可以这样使用结构体了：`XXRect rect2 = {1, 2, 3, 4}; NSValue *value2 = @(rect2);`。     

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
作用于变量，当变量的作用域结束是，会调用指定的函数，语法为`__attribute__((cleanup))`。该属性可以作用于变量，block，例如：   
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


#### Clang Attributes        
Clang Attributes是Clang提供的一种源码注解，方便开发者向编译器表达某种要求，

**availability**  
指明API(变量/方法)版本的变更，语法为`__attribute__((availability(macosx,introduced=m,deprecated=n)))`，m为引入的版本，n为废弃版本，在Cocoa的头文件中可以经常见到。例如：    
```
#define CF_DEPRECATED_IOS(_iosIntro, _iosDep, ...)    __attribute__((availability(ios,unavailable,introduced=_iosIntro,deprecated=_iosDep,message="" __VA_ARGS__)))
```
其他的参数为: obsoleted为移除的版本，unavailable为不可用，message为警告信息。系统宏`NS_AVAILABLE`、`NS_CLASS_AVAILABLE`的展开即为该属性。  

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

#### iOS中的宏  

##### **系统宏**  
a. NS_ASSUME_NONNULL  
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

b. 可用性的宏标记    

```
NS_CLASS_AVAILABLE   // 类之前
NS_AVAILABLE(_mac, _ios) // 方法之后
NS_DEPRECATED(_macIntro, _macDep, _iosIntro, _iosDep, ...) // 方法之后
OBJC_SWIFT_UNAVAILABLE("use 'xxx' instead") // 针对swift特殊说明取代它的类和方法
```

##### **常量/表达式宏**   
例如:  
```
#define kTabBarH 49.0
#define kScreenH [UIScreen mainScreen].bounds.size.height
#define MKAssertMainThread() NSAssert([NSThread isMainThread], @"This method must be called on the main thread")  
```

**函数宏**  
例如：  
```
#ifndef YYSYNTH_DYNAMIC_PROPERTY_OBJECT
#define YYSYNTH_DYNAMIC_PROPERTY_OBJECT(_getter_, _setter_, _association_, _type_) \
- (void)_setter_ : (_type_)object { \
    [self willChangeValueForKey:@#_getter_]; \
    objc_setAssociatedObject(self, _cmd, object, OBJC_ASSOCIATION_ ## _association_); \
    [self didChangeValueForKey:@#_getter_]; \
} \
- (_type_)_getter_ { \
    return objc_getAssociatedObject(self, @selector(_setter_:)); \
}
#endif
```  
还有一种变参宏，例如：    
`#define YYAssertNil(condition, description, ...) NSAssert(!(condition), (description), ##__VA_ARGS__)`  
在宏定义中`#`可以连接两个字符串或者将变量转化为C语言字符串，`##`则常用与连接，另外在逗号和__VA_ARGS__之间的双井号，除了拼接前后文本之外，还有一个功能，那就是如果后方文本为空，那么它会将前面一个逗号吃掉。再例如：
`#define weakify(object) autoreleasepool{} __weak __typeof__(object) weak##_##object = object;`  

文章分别总结了常用的attribute属性和一些宏，如有相关问题，请评论指正。后续有其他内容会陆续补充到文章中。  

###### 参考文档  
1.[llvm __attribute__](http://releases.llvm.org/3.8.0/tools/clang/docs/AttributeReference.html#assume-aligned-gnu-assume-aligned)
2.[__attribute__](http://nshipster.com/__attribute__/)
3.[__attribute__((cleanup))](http://honglu.me/2016/03/04/Cocoa%E4%B8%AD%E5%B8%B8%E7%94%A8%E7%9A%84%E5%AE%8F%E5%AE%9A%E4%B9%89/)  
