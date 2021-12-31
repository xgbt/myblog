---
title: "SOLID原则简介"
date: 2021-12-11T21:12:53+08:00
draft: false
tags: [设计模式]
categories: [笔记] 
---

# 1. 单一职责原则

单一职责原则 **S**ingle Responsibility Principle

> *"There should never be more than one reason for a class to change." — Robert Martin.*
>
> 一个类应该只有一个发生变化的原因

**一个类应该专注做一件事**

SRP原则具体指一个类应该专注做一件事，或者只有一个改变的理由。这个类里不一定只有一种方法，但是所有方法应该指向同一个目的。

# 2. 开闭原则

开闭原则 **O**pen Closed Principle

> *"Software entities (modules, classes, functions, etc.) should be open for extension, but closed for modification." — Robert Martin.*
>
> 一个软件实体，如类、模块和函数应该对扩展开放，对修改关闭

**使用继承 或者 组合改变类的功能。**

遵循 OCP 能帮主我们更简单地改变类的功能，并且避免在改变的时候破坏现有功能。OCP 还使我们考虑类中可能发生变化的地方，有助于选择设计所需要的正确抽象（the right abstractions）。

# 3. 里氏替换原则

里氏替换原则 **L**iskov Substitution Principle

> *"Functions that use pointers or references to base classes must be able to use objects of derived classes without knowing it." — Robert Martin.*
>
> 所有引用基类的地方必须能透明地使用其子类的对象

**派生类在代替基类使用时应该表现地更好。**

LSP 确保继承正确使用。如果一个覆盖的方法什么都不做或者只是抛出异常，那么可能违反了LSP。

# 4. 接口隔离原则

接口隔离原则 Interface Segregation Principle

> "Clients should not be forced to depend upon interfaces that they do not use." — Robert Martin.
>
> 不应该强迫客户依赖他们不使用的接口

**保证接口小而精**

ISP 旨在保持接口（和抽象类）很小并且仅限于非常特定的需求。如果您有一个臃肿的接口，那么您就会对使用该接口的人施加巨大的实现负担。

# 5. 依赖倒置原则

依赖倒置原则 **D**ependency Inversion Principle

> "High level modules should not depend upon low level modules. Both should depend upon abstractions."
>
> "Abstractions should not depend upon details. Details should depend upon abstractions."
>
> — Robert Martin.
>
> 上层模块不应该依赖底层模块，它们都应该依赖于抽象。
> 抽象不应该依赖于细节，细节应该依赖于抽象。

**依赖于抽象，而不是具体的实现。**

DIP 告诉我们如果一个类依赖于其他类，它应该依赖抽象而不是它们的具体类型。主要思想是，将程序员都写的类隔离在所依赖的抽象之后，这样即使将来所有的抽象都发生了变化，那类仍然是安全的，有助于保持低耦合，使得代码更容易修改。

DIP 另一面涉及了分层应用程序中，高级和低级模块之间的依赖关系。例如，数据库访问类不应依赖于用于显示该数据的 UI 表单。相反，UI 应该依赖于数据库访问类的抽象。

当DIP 与 [IoC container](https://www.ledjonbehluli.com/posts/wash-tunnel/ioc_container/) & [Dependency Injection](https://www.ledjonbehluli.com/posts/wash-tunnel/dependency_injection/)相结合时，DIP 会变得非常有用。

> *DIP is about shape, IoC is about direction, and DI is about wiring.*



# 参考文献

- https://www.ledjonbehluli.com/posts/wash-tunnel/introduction/
