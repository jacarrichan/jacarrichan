title: 设计模式及其在java中的体现
date: 2017-01-18 10:34:59
tags: [设计模式,jdk]
---


#  分类

GoF的设计模式一共23个，可以分为3大类：创建型、结构型和行为型

##. 创建型
* 简单工厂（Simple Factory）、工厂方法（Factory Method）、抽象工厂（Abstract Factory）、单例（Singleton）、构造者（Builder）和原型（Prototype）

 工厂系列的3个设计模式，它们都主要是针对软件设计中的“开放-封闭”原则，即程序应该对扩展开放，对修改封闭。特别是当我们的程序采用XML+反射的方式来创建对象时，工厂模式的威力就完全展现出来了，这时我们可以通过维护配置文件的方式，来控制程序的逻辑。

## 结构型
* 适配器（Adapter）、装饰（Decorator）、桥接器（Bridge）、享元（FlyWeight）、门面（Facade）、合成（Composite）以及代理（Proxy）模式。


## 行为型
* 职责链（Chain of Responsibility）、命令（Command）、解释器（Interperter）、迭代（Iterator）、中介者（Mediator）、备忘录（Memento）、观察者（Observer）、状态（State）、策略（Strategy）、模板（Template）和访问者（Visitor）。


# 6种原则：
　　**S：单一职责原则**（Single Responsibility Principle, SRP），一个类只能有一个原因使其发生改变，即一个类只承担一个职责。

　　**O：开放-封闭原则**（Open-Close Principle, OCP），这里指我们的设计应该**针对扩展开放，针对修改关闭**，即尽量以扩展的方式来维护系统。

　　**L：里氏替换原则**（Liskov Subsititution Principle, LSP），它表示我们可以在代码中使用任意子类来替代父类并且程序不受影响，这样可以保证我们使用“继承”并没有破坏父类。

　　**I：接口隔离原则**（Interface Segregation Principle, ISP），客户端不应该依赖于它不需要的接口，**两个类之间的依赖应该建立在最小接口的基础上**。这条原则的目的是为了让那些使用相同接口的类只需要实现特定必要的一组方法，而不是大量没用的方法。

　　**D：依赖倒置原则**（Dependence Inversion Principle, DIP），高层模块不应该依赖于低层模块，两者应该都依赖于抽象；**抽象不依赖于细节，而细节应该依赖于抽象**。这里主要是提倡“面向接口”编程，而非“面向实现”编程。

　　**L：迪米特法则**(Law of Demete),也叫最少知识原则，一个对象应该对其他对象有最少的了解，或者“只和朋友交流” ，用设计行话就是“**高内聚，低耦合**”，减少类之间的相互依赖，修改系统的某一部分的时候，不会影响到其他部分，使系统有更好的维护性. 


# 简单总结：

### 工厂方法（Factory）

* 意图：定义一个用户创建对象的接口，让子类去决定具体使用哪个类。
* 适用场合：1）类不知道它所要创建的对象的类信息；2）类希望由它的子类来创建对象。
* JDK中的体现：Collection.iterator方法

### 抽象工厂（Abstract factory）

* 意图：提供一个创建一系列相关或者相互依赖的对象的接口，而无须指定它的具体实现类。
* 适用场合：1）系统不依赖于产品是如何实现的细节；2）系统的产品族大于1，而在运行时刻只需要某一种产品族；3）属于同一个产品族的产品，必须绑在一起使用；4）所有的产品族，可以抽取公共接口
* JDK中的体现：java.sql包

### 单例（Singleton）

* 意图：保证一个类只有一个实例，并且在系统全局范围内提供访问切入点。
* 适用场合：各种“工厂类”
* JDK中的体现：Runtime类,Emun类

### 构造者（Builder）

* 意图：**简化复杂对象的创建**，将复杂对象的构造与表示相分离，使得同样的构造过程可以产生不同的复杂对象。
* 适用场合：1）需要创建的对象有复杂的内部结构；2）对象的属性之间相互依赖，创建时前后顺序需要指定。
* JDK中的体现：java.sql.PreparedStatement

### 原型（Prototype）

* 意图：用原型实例指定创建对象的种类，并通过复制原型实例得到对象。使用自己的实例创建另一个实例。有时候，创建一个实例然后再把已有实例的值拷贝过去，是一个很复杂的动作。所以，使用这个模式可以避免这样的复杂性。
* 适用场合：1）系统不关心对象创建的细节；2）要实例化的对象的类型是动态加载的；3）类在运行过程中的状态是有限的。
* JDK中的体现：java.lang.Object#clone()，java.lang.Cloneable

### 适配器（Adapter）

* 意图：将一个类的接口转换成用户希望的另一个接口。
* 适用场合：系统需要使用现有类的功能，但接口不匹配
* JDK中的体现：java.io.OutputStreamWriter(OutputStream),java.util.Arrays#asList()

### 装饰（Decorator）

* 意图：动态的为对象添加额外职责
* 适用场合：1）需要添加对象职责；2）这些职责可以动态添加或者取消；3）添加的职责很多，从而不能用继承实现。
* JDK中的体现：java.io.BufferedInputStream(InputStream)

### 桥接器（Bridge）

* 意图：将抽象部分与实现部分分离，从而使得它们可以独立变化
* 适用场合：1）系统需要在组件的抽象化角色与具体化角色之间增加更多的灵活；2）角色的任何变化都不应该影响客户端；3）组件有多个抽象化角色和具体化角色
* JDK中的体现：AWT ,JDBC

### 享元（Flyweight）

* 意图：运用共享技术支持大量细粒度的对象
* 适用场合：1）系统中有大量对象；2）这些对象占据大量内存；3）对象中的状态可以很好的区分为外部和内部；4）可以按照内部状态将对象分为不同的组；5）对系统来讲，同一个组内的对象是不可分辨的
* JDK中的体现：java.lang.Integer#valueOf(int)，java.lang.Byte#valueOf(byte)

### 门面（Facade）
* 意图：为系统的一组接口提供一个一致的界面
* 适用场合：1）为一个复杂的接口提供一个简单界面；2）保持不同子系统的独立性；3）在分层设计中，定义每一层的入口
* JDK中的体现：java.lang.Class

### 合成（Composite）

* 意图：将对象组装成树状结构以表示“部分-整体”的关系
* 适用场合：1）系统中的对象之间是“部分-整体”的关系；2）用户不关心“部分”与“整体”之间的区别
* JDK中的体现：java.util.Map#putAll(Map),java.util.List#addAll(Collection)

### 代理（Proxy）

* 意图：为其他对象提供一种代理以控制对该对象的访问
* 适用场合：对象无法直接访问（远程代理）
* JDK中的体现：java.lang.reflect.Proxy,RMI

### 职责链（Chain of responsibility）

* 意图：对目标对象实施一系列的操作，并且不希望调用双方和操作之间有耦合关系
* 适用场合：1）输入对象需要经过一系列处理；2）这些处理需要在运行时指定；3）需要向多个操作发送处理请求；4）这些处理的顺序是可变的
* JDK中的体现：java.util.logging.Logger#log(),javax.servlet.Filter#doFilter()

### 命令(Command)

* 意图：对一类对象公共操作的抽象
* 适用场合：1）调用者同时和多个执行对象交互；2）需要控制调用本身的生命周期；3）调用可以取消
* JDK中的体现：java.lang.Runnable

### 观察者（Observer）

* 意图：定义对象之间一种“一对多”的关系，当一个对象发生改变时，所有和它有依赖关系的对象都会得到通知
* 适用场合：1）抽象模型有两部分，其中一部分依赖于另一部分；2）一个对象的改变会导致其他很多对象发生改变；3）对象之间是松耦合
* JDK中的体现：java.util.EventListener,javax.servlet.http.HttpSessionBindingListener

### 访问者（Visitor）

* 意图：对一组不同类型的元素进行处理
* 适用场合：1）一个类型需要依赖于多个不同接口的类型；2）需要经常为一个结构相对稳定的对象添加新操作；3）需要用一个独立的类型来组织一批不相干的操作，使用它的类型可以根据应用需要进行定制
* JDK中的体现：javax.lang.model.element.Element 和javax.lang.model.element.ElementVisitor

*** 责任链、观察者 访问者的区别和联系***

可以简单理解，责任链和访问者都是观察者的变种。观察者通过事件机制通知到各个节点，如果节点只有一个处理者即为责任链模式，如果都处理，则是观察者模式，如果有条件处理，则是访问者模式。

### 模板（Template）

* 意图：定义一个操作步骤的方法骨架，而将其中一些细节的实现放到子类中
* 适用场合：1）可以抽取方法骨架；2）控制子类的行为，只需要实现特定细节
* JDK中的体现：java.util.Collections#sort(),java.io.InputStream#skip(),java.io.InputStream#read()

### 策略（Strategy）

* 意图：对算法族进行封装
适用场合：1）完成某项业务有多个算法；2）算法可提取公共接口
* JDK中的体现：ThreadPoolExecutor中的四种拒绝策略

### 解释器（Interpreter）

* 意图：应用或对象与用户狡猾时，采取最具实效性的方式完成
* 适用场合：1）针对对象的操作有规律可循；2）在执行过程中，对效率要求不高，但对灵活性要求很高
* JDK中的体现：java.util.regex.Pattern

### 迭代（Iterator）

* 意图：提供一种方法， 来顺序访问集合中的所有元素
* 适用场合：1）访问一个聚合对象的内容，而不必暴露其内部实现；2）支持对聚合对象的多种遍历方式；3）为遍历不同的聚合对象提供一致的接口
* JDK中的体现：terator、Enumeration接口

### 中介者（Mediator）

* 意图：避免大量对象之间的紧耦合
* 适用场合：1）有大量对象彼此依赖（M：N）；2）某个类型要依赖于很多其他类型
* JDK中的体现：java.util.concurrent.Executor#execute(),java.util.concurrent.ExecutorService#submit(),java.lang.reflect.Method#invoke()

### 备忘录（Memento）

* 意图：希望备份或者恢复复杂对象的部分属性
* 适用场合：1）对象的属性比较多，但需要备份恢复的属性比较少；2）对象的状态是支持恢复的
* JDK中的体现：java.util.Date,java.io.Serializable,Null Object:

### 状态（State）

* 意图：管理对象的多个状态
* 适用场合：1）对象的行为依赖于当前状态；2）业务处理过程存在多个分支，而且分支会越来越多
* JDK中的体现：java.util.Iterator



----

抽个时间把[并发设计模式](https://en.wikipedia.org/wiki/Software_design_pattern#Concurrency_patterns 'Concurrency_patterns')学学

