# Item13. 使类和成员的可访问性最小化



模块之间通过 api 通信，一个模块不需要知道其他模块内部的工作情况，这个概念称为 **信息隐藏**（**或封装**）。



解除系统和各模块耦合，模块可独立开发、测试、优化、使用、理解、修改。加快系统开发速度，减轻维护负担。



Java 提供的 **信息隐藏** 机制：访问控制机制（决定了类、接口、成员）的可访问性。实体的可访问性取决于其声明的位置，以及声明中存在哪些访问修饰符（private，protected和public）。 正确使用这些修饰符对信息隐藏至关重要。



## 让每个类或成员尽可能地不可访问

第一规则很简单：**让每个类或成员尽可能地不可访问**



如果一个包级私有顶级类或接口只被一个类使用，那么可以考虑这个类作为使用它的唯一类的私有静态嵌套类。



但是，**减少不必要的公共类的可访问性**要比包级私有的顶级类更重要：**公共类是包的API的一部分**，而包级私有的顶级类已经是这个包实现的一部分了。



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-11-13-21-53-36_r69.png)



## 非 public 尽量 private，除非同一个包的其他类要访问

当仔细设计了类的公有 api 之后，应该把所有其他成员都变成 `private`，只有当同一个包下另一个类想要访问这个类的成员的时候，**才应该删除 `private`**，把访问性变成 `package-private`（即缺省的访问级别）。如果你发现自己经常这样做，你应该重新检查你的系统的设计，看看另一个分解可能产生更好的解耦的类。 也就是说，**私有成员和包级私有成员都是类实现的一部分，通常不会影响其导出的API。**但是，如果类实现`Serializable`接口，则这些属性可以**“泄漏”到导出的API**中。





## 受保护成员应该尽量少用

对于`public class`的成员，当访问级别从 `package-private` 到 `protected` 级时，会大大增加其可访问性。 **受保护（protected）的成员是类导出的API的一部分，并且必须永远支持。** 此外，导出类的受保护成员表示对实现细节的公开承诺）。 **受保护成员应该尽量少用。**





---



## 子类中的访问级别不允许低于父类中的访问级别

**子类中的访问级别不允许低于父类中的访问级别**，这样才可以确保任何使用父类的地方可以使用子类的实例（防止父类能访问，子类反而不能访问）。否则会编译出错。这个规则的一个特例是，如果一个类实现了一个接口，那么接口中的所有类方法都必须在该类中声明为`public`，因为接口中的所有方法都隐含着公有的访问级别。



---



##不能为了测试把private提升到包导出的API一部分

为了测试容易，提升访问级别是可以接受的，比如 `private` 变成 `package-private`，但超过这个就不能接受了。



幸运的是也没必要这么做，因为可以让测试作为被测试的包的一部分来运行，从而能够访问 `package-private` 的元素。



## 实例域决不能是 public 的、静态域也是



如果域是**非 final** 的，或者是个**指向可变对象的 final 引用**，那么一旦变成公有，就放弃了存储在这个域中的值进行限制的能力；意味着放弃了强制这个域不可变的能力。同时这个域被修改的时候，也失去了对它采取行动的能力。**因此包含公有可变域的类并不是线程安全的。**即使域  final 的，并且引用不可变对象，当这个域变 `public `时，也就放弃了 “切换到一种新的内部数据表示法” 的灵活性。



静态域也是同样的建议。只有一种例外：**常量构成了类提供的整个抽象的一部分**，可以通过公有静态 final 域来暴露这些常量，按惯例这种于由大写字母组成，单词直接用下划线隔开。而且这些域要么包含基本类型的值，要么指向不可变对象的引用（否则引用的对象能被修改）。



请注意，非零长度的数组总是可变的，所以**类具有公共静态final数组属性，或返回这样一个属性的访问器，这几乎总是错误的**。 如果一个类有这样的属性或访问方法，客户端将能够修改数组的内容。 这是安全漏洞的常见来源：



```java
// Potential security hole!  错误
public static final Thing[] VALUES = { ... };
```





要小心这样的事实，一些 IDE **生成的访问方法返回对私有数组属性的引用**，导致了这个问题。 有两种方法可以解决这个问题。 **你可以使公共数组私有并添加一个公共的不可变列表：**



```java
private static final Thing[] PRIVATE_VALUES = { ... };

public static final List<Thing> VALUES = Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));
```



**或者，可以将数组设置为private，并添加一个返回私有数组拷贝的公共方法：**



```java
private static final Thing[] PRIVATE_VALUES = { ... };

public static final Thing[] values() {
    return PRIVATE_VALUES.clone(); // 访问数组只能用这个方法，返回一个数组的拷贝
}
```



要在这些方法之间进行选择，请考虑客户端可能如何处理返回的结果。 哪种返回类型会更方便？ 哪个会更好的表现？





---



## Java 9 额外的隐式访问级别



在Java 9中，作为模块系统（module system）的一部分引入了**两个额外的隐式访问级别**。**模块包含一组包，就像一个包包含一组类一样。**模块可以通过模块声明中的导出（export）声明显式地导出某些包(这是module-info.java的源文件中包含的约定)。模块中的未导出包的公共和受保护成员在模块之外是不可访问的；在模块中，可访问性不受导出（export）声明的影响。使用模块系统允许你在模块之间共享类，而不让它们对整个系统可见。在未导出的包中，公共和受保护的公共类的成员会产生两个隐式访问级别，这是普通公共和受保护级别的内部类似的情况。这种共享的需求是相对少见的，并且可以通过重新安排包中的类来消除。

**与四个主要访问级别不同，这两个基于模块的级别主要是建议（advisory）**。 如果将模块的JAR文件放在应用程序的类路径而不是其模块路径中，那么模块中的包将恢复为非模块化行为：包的公共类的所有公共类和受保护成员都具有其普通的可访问性，不管包是否由模块导出。 新引入的访问级别严格执行的地方是JDK本身：Java类库中未导出的包在模块之外真正无法访问。

对于典型的Java程序员来说，不仅程序模块所提供的访问保护存在局限性，而且在本质上是很大程度上建议性的；为了利用它，你必须把你的包组合成模块，在模块声明中明确所有的依赖关系，重新安排你的源码树层级，并采取特殊的行动来适应你的模块内任何对非模块化包的访问。 现在说模块是否会在JDK之外得到广泛的使用还为时尚早。 与此同时，除非你有迫切的需要，否则似乎最好避免它们。



## 总结



总而言之，应该尽可能地减少程序元素的可访问性（在合理范围内）。 在仔细设计一个最小化的公共API之后，你应该防止任何散乱的类，接口或成员成为API的一部分。 除了作为常量的公共静态final属性之外，公共类不应该有公共属性。 确保`public static final`属性引用的对象是不可变的。



