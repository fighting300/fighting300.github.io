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




##### 参考文档
1.[https://developer.apple.com/library/content/documentation/...](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Initialization.html)  
2.https://www.pupboss.com/swift-init-modifiers/
