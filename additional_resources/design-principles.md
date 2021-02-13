# 设计原则

## 常见设计原则概述
---

## [SOLID](https://en.wikipedia.org/wiki/SOLID)

- [单一权责原则Single Responsibility Principle (SRP)](https://en.wikipedia.org/wiki/Single-responsibility_principle):
  一个类只应有一种责任，只有对软件中特定的一部分修改时才会影响到类。
- [开闭原则Open/Closed Principle (OCP)](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle):
  软件应该对扩展开放，但是对修改封闭。
- [里氏替换原则Liskov Substitution Principle (LSP)](https://en.wikipedia.org/wiki/Liskov_substitution_principle):
  子类可以扩展父类的功能，但不能改变父类原有的功能
- [接口隔离原则Interface Segregation Principle (ISP)](https://en.wikipedia.org/wiki/Interface_segregation_principle):
  多个专一功能的接口比一个泛用的接口要好。
- [依赖倒置原则Dependency Inversion Principle (DIP)](https://en.wikipedia.org/wiki/Dependency_inversion_principle):
  应该依赖抽象而不是依赖于细节。

## [DRY (Don’t Repeat Yourself)](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)

在一个系统中，每一个知识都必须有一个单一、明确、权威的表示。

## [KISS原则KISS principle](https://en.wikipedia.org/wiki/KISS_principle)

绝大多数系统简单时比复杂时工作的要好。因此简单性是设计中的关键目标，并且应该避免不必要的复杂性。

## [迪米特法则Law of Demeter (LoD)](https://en.wikipedia.org/wiki/Law_of_Demeter)

一个实体应该尽可能少的与任何其他的结构或者特性（包括子组件）发生关系，符合“信息隐藏”的原则。

## [契约式设计Design by contract (DbC)](https://en.wikipedia.org/wiki/Design_by_contract)

软件设计者应该为软件组件定义规范、准确和可验证的接口，扩展了抽象数据类型的平凡定义，包括前置条件、后置条件和不变量。

## [封装Encapsulation](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming))

将数据与对该数据进行操作的方法捆绑在一起，或者限制对对象某些组件的直接访问。封装用于隐藏类中结构体对象的值或状态，防止未经授权地直接访问它们。

## [命令查询分离原则Command-Query-Separation(CQS)](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)

函数不应该产生抽象的副作用，只允许命令（过程）产生副作用——Bertrand Meyer:《面向对象软件构造》



## [最小惊奇原则Principle of least astonishment (POLA)](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)

系统的组件应该像人们期望的那样工作，而不应该给用户一个惊奇。

## 语言模块单元Linguistic-Modular-Units

模块必须与使用的语言单元相符合——Bertrand Meyer:《面向对象软件构造》

## 自文档Self-Documentation

一个模块的设计者应该努力使所有关于该模块的信息成为模块本身的一部分——Bertrand Meyer:《面向对象软件构造》

## 统一访问原则Uniform-Access

一个模块提供的所有服务都应该通过一个统一的符号来提供，而这个符号并不表明它们是通过存储还是通过计算来实现的。——Bertrand Meyer:《面向对象软件构造》

## 单一选择Single-Choice

每当软件系统必须支持一组备选方案时，系统中应该只有一个模块知道它们的底细。——Bertrand Meyer:《面向对象软件构造》

## 存储闭包Persistence-Closure

当存储一个对象时，必须将其所依赖的部分一起存储。每当检索机制检索以前存储的对象时，它还必须检索该对象的尚未检索到的所有依赖项。——Bertrand Meyer:《面向对象软件构造》