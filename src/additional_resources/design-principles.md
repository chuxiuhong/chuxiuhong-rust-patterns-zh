# 设计原则

## 常见设计原则概述

---

## [SOLID](https://en.wikipedia.org/wiki/SOLID)

- [单一职责原则 (Single Responsibility Principle, SRP)](https://en.wikipedia.org/wiki/Single-responsibility_principle)：
  一个类应该只有一个职责，即软件规格说明的某一部分的变化应该只影响该类的规格说明。

- [开放/封闭原则 (Open/Closed Principle, OCP)](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle)：
  "软件实体应该对扩展开放，对修改封闭。"

- [里氏替换原则 (Liskov Substitution Principle, LSP)](https://en.wikipedia.org/wiki/Liskov_substitution_principle)：
  "程序中的对象应该可以被其子类型的实例替换，而不会影响程序的正确性。"

- [接口隔离原则 (Interface Segregation Principle, ISP)](https://en.wikipedia.org/wiki/Interface_segregation_principle)：
  "多个特定于客户端的接口要好于一个通用接口。"

- [依赖倒置原则 (Dependency Inversion Principle, DIP)](https://en.wikipedia.org/wiki/Dependency_inversion_principle)：
  "应该依赖于抽象，而不是具体实现。"

## [组合复用原则 (Composite Reuse Principle, CRP) 或 组合优于继承](https://en.wikipedia.org/wiki/Composition_over_inheritance)

"类应该通过组合（包含实现所需功能的其他类的实例）来实现多态行为和代码复用，而不是通过继承基类或父类。" - Knoernschild, Kirk (2002). Java Design - Objects, UML, and Process

## [DRY 原则 (Don't Repeat Yourself)](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)

"系统中的每一项知识都必须有一个单一的、明确的、权威的表示"

## [KISS 原则](https://en.wikipedia.org/wiki/KISS_principle)

大多数系统在保持简单而非复杂时运行效果最佳；因此，简单性应该是设计中的一个关键目标，应避免不必要的复杂性

## [迪米特法则 (Law of Demeter, LoD)](https://en.wikipedia.org/wiki/Law_of_Demeter)

一个对象应该对其他对象（包括其子组件）的结构或属性知道得越少越好，这符合"信息隐藏"原则

## [契约式设计 (Design by contract, DbC)](https://en.wikipedia.org/wiki/Design_by_contract)

软件设计者应该为软件组件定义正式的、精确的和可验证的接口规范，这些规范在普通抽象数据类型的基础上扩展了前置条件、后置条件和不变量

## [封装 (Encapsulation)](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming))

将数据与操作这些数据的方法捆绑在一起，或限制直接访问对象的某些组件。封装用于在类中隐藏结构化数据对象的值或状态，防止未经授权的直接访问

## [命令查询分离 (Command-Query-Separation, CQS)](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)

"函数不应产生抽象的副作用...只有命令（过程）才允许产生副作用。" - Bertrand Meyer: Object-Oriented Software Construction

## [最小惊讶原则 (Principle of least astonishment, POLA)](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)

系统的组件应该以大多数用户期望的方式运行。其行为不应该让用户感到惊讶或意外

## 语言模块单元原则 (Linguistic-Modular-Units)

"模块必须对应于所使用语言中的语法单元。" - Bertrand Meyer: Object-Oriented Software Construction

## 自文档化原则 (Self-Documentation)

"模块设计者应努力使关于模块的所有信息都成为模块本身的一部分。" - Bertrand Meyer: Object-Oriented Software Construction

## 统一访问原则 (Uniform-Access)

"模块提供的所有服务都应该通过统一的表示法来访问，这种表示法不应该暴露它们是通过存储还是通过计算来实现的。" - Bertrand Meyer: Object-Oriented Software Construction

## 单一选择原则 (Single-Choice)

"当软件系统必须支持一组替代方案时，系统中应该只有一个模块知道它们的完整列表。" - Bertrand Meyer: Object-Oriented Software Construction

## 持久性闭包原则 (Persistence-Closure)

"当存储机制存储一个对象时，它必须同时存储该对象的依赖对象。当检索机制检索先前存储的对象时，它也必须检索该对象尚未检索的任何依赖对象。" - Bertrand Meyer: Object-Oriented Software Construction
