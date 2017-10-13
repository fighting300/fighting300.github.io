---
title: Swift语法特性
tags: Swift
categories: iOS
date: 2016-11-01 16:32:01
---

最近在写swift的过程中，发现有很多特性使用的不太熟练，而且有些新特性看过之后很容易忘记，所以该文章记录一些Swift语法的新特性。(本文持续更新)  

#### 1. 初始化

**Designated initializers**
即最常用的初始化方法init()，默认不加修饰符的init方法都需要在方法中保证所有的非Optional实例变量被赋值初始化，其子类也会隐式或显式调用父类的designated初始化方法。  
```
  class House {
    let door: Int
    init(door: Int) {
        self.door = door
    }
  }
  class Room: House {
    let table: Int
    override init(door: Int) {
        self.table = 0
        super.init(door: door)
    }
  }  
```

这儿注意如果子类有新增的属性的话，必须调用override重写父类init方法。如果没有的话，则可以不重写，直接使用父类的Designated初始化方法创建子类对象。推荐在init使用self给变量赋值。
<!--more-->  

**Convenience initializers**  
即补充的初始化方法，可有可无。需要在init前加convenience关键字，必须调用同一类中的`Designated initializers`初始化来完成设置，另外该初始化方法不能被子类重写或者从子类中以super的方式被调用。  

```
  class House {
    let door: Int
    var window: Int?
    init(door: Int) {
        self.door = door
    }
    convenience init(window: Int) {
        self.init(door: 0)
    }
  }
  class Room: House {
    let table: Int
    override init(door: Int) {
        self.table = 0
        super.init(door: door)
    }
  }
```

如果在子类中实现了父类Conveniences所调用的Designated初始化方法的话，子类可以直接使用父类的Convenience方法。如：`let room = Room(window: 1)`   

**required**
required关键词可以强制限制子类必须重写Designated初始化方法，这样可以保证相关依赖的convenience方法在子类中可以使用。如果子类中没有init方法，Designated依旧会被隐式调用，并不会报错。如果想要在子类中使用父类的Convenience或者init方法，最好给init方法加上required关键字。

```
  class House {
      let door: Int
      var window: Int?
      required init(door: Int) {
          self.door = door
      }
  }
  class Room: House {
      var table: Int?
      required init(door: Int) {
          self.table = 1
          super.init(door: door)
      }
  }
```

#### 2. 访问控制

在Swift中，访问修饰符有以下5种，有点类似Java的访问控制符。其中fileprivate和open是swift3才支持的新特性，由于原有的private和public修饰符可能存在语义上不足，所以新加了两种新的访问控制权限，这样达到更安全的效果。

**fileprivate**  
所修饰的属性和方法在当前的swift源文件里可以访问，这样基于文件做了访问控制。  
**open**  
可以被任何人使用，包括重写和继承。  
**private**
所修饰的属性或者方法只能在当前类中访问，这样做到了真正私有，离开当前类或结构体的作用域就无法访问。Swift4中private的变量在其extension中可以可被读取。
**internal**
默认访问级别，可以忽略不写。其所修饰的属性/方法在源码所在的整个模块都可以访问。如果是框架/库代码，则框架/库内部可以访问，由外部引用是，则不可以访问。
**public**
可以被任何人访问，但是在其他module中不可以被重写(override)或继承，在module内可以被重写或者继承。   
**final**
修饰的类或属性不应该被继承或修改。  

这样当前访问权限的优先排序依次为：  
`open -> public -> internal -> fileprivate -> private`

#### 3. inout 传址调用
inout关键词用来将一个值类型以引用方式传递，这样通过函数可以改变函数外面的值，Swift的数据类型中Int、Float、Bool、Character、Array、Set、enum、struct全是值类型，只有class是引用类型。在调用函数时，变量名字前面用&符号修饰表示，具体使用方法如下：

```
var value = 50
func increment(_ value: inout Int, length: Int = 10) {
    value += length
}
func getResult() {
    increment(&value)
    print(value)
}
```

另外inout参数不能有默认值，且可变参数不能用inout标记。一个参数一旦被inout修饰，就不能再被var和let修饰了。

#### 4. 错误处理 -- guard 提前退出
guard和if类似，只是作用相反，表示如果条件不满足时退出当前block，并且强制在else中用return来退出函数、continue、throw或break退出循环，或者用一个类似fatalError()的@noreturn函数来退出，以离开当前上下文。
guard可以快速检查当前可能出现的错误，并立即跳出当前流程。这样减少了if判断的嵌套，同时还可以转换一个optional值，之后该值可以直接使用。

```
guard let count = imageNamesList?.count, count > 0 else {
    return
}
for imageName in imageNamesList {
    guard let image = UIImage(named: imageName)
        else { continue }
    // do something with image
}
```

guard的else语句中，除了简单的提前退出语句，或者一些诊断日志外，不应该有其他的逻辑。而一些未完成工作的清理或者资源释放应该使用defer来完成，通常else中的代码一般不要多余3行。
通过guard的方式，可以确保程序在可知的情况下退出。如果判断条件仅是简单的布尔表达式的话，可以使用`precondition`，`precondition(internet.node == 100, "Not enough node in the internet")`。或者可以使用断言`assertionFailure`，这样在正式发布的时候，应用不会crash掉，开发和测试期间也能找到bug位置，如果判断条件仅是简单的布尔表达式的话，直接使用`assert(condition)`即可  

```
guard let node = internet.node else {
    fatalError("OMG ran out of node!")
    assertionFailure("Huh, no dogs")
}
```

#### 5. 错误处理 -- defer 延迟执行

声明一个block来包含一段代码块，将会在当前作用域结束的时候(一般指return时)被调用。通常用来对当前的代码进行清理工作，比如关闭打开的目录、文件等。语法为：`defer { ... }`
defer的block执行顺序和书写顺序是相反的，即先进后出，先定义的后执行。如以下代码中，`defer 0`最后输出:

```
func doDeferPrint() {
    print("first")

    defer {
        print("defer 0")
    }
    print("second")
    defer {
        print("defer 1")
    }
}
```

但是defer的使用不应该降低代码的可读性，应该严格遵循defer在整个程序最后运行以释放已申请资源的原则，其他类型的使用会让代码乱成一团。例如以下的代码不被推荐:
```
postfix func ++(x: inout Int) -> Int {
    defer { x += 1 }
    return x
}
```


#### 6.precondition  discardableResult

#### 7.协议

##### 参考文档
1.[https://developer.apple.com/library/content/documentation/...](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Initialization.html)  
2.https://www.pupboss.com/swift-init-modifiers/
3.http://swift.gg
