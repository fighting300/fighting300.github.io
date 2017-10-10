---
title: Swift语法特性
tags: Swift
categories: iOS
date: 2017-06-01 16:32:01
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
所修饰的属性或者方法只能在当前类中访问，这样做到了真正私有，离开当前类或结构体的作用域就无法访问。  
**internal**
默认访问级别，可以忽略不写。其所修饰的属性/方法在源码所在的整个模块都可以访问。如果是框架/库代码，则框架/库内部可以访问，由外部引用是，则不可以访问。
**public**
可以被任何人访问，但是在其他module中不可以被重写(override)或继承，在module内可以被重写或者继承。   
**final**
修饰的类或属性不应该被继承或修改。  

这样当前访问权限的优先排序依次为：  
`open -> public -> internal -> fileprivate -> private`



##### 参考文档
1.[https://developer.apple.com/library/content/documentation/...](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Initialization.html)  
2.https://www.pupboss.com/swift-init-modifiers/
