# Item 4. 通过私有构造器强化不可实例化能力

有时候需要编写不可实例化的类，只使用里面的静态方法、静态域。这些类的名声很不好，因为有人在面向对象的语言中滥用这样的类来编写过程化的程序，尽管如此，他们有其特有的用处。


比如以 `java.lang.Math` 或者 `java.util.Arrays` 的方式，把基本类型的值或数组类型的相关方法组织其阿里，也可以通过 `java.util.Collections` 把实现特定接口的对象上的静态方法组织起来。最后还可以利用这种类，把 final 类上的方法组组织起来，以取代扩展该类的做法。

这样的 **工具类** 不希望被实例化，实例对它没有任何意义。显然缺少显示构造器的情况下，编译器自动为其提供一个公有、无参的缺省构造器(default constructor)，那么用户可以用来实例化。

企图通过将类做成抽象类来强制该类不可实例化，这是行不通的（虽然抽象类也可以使用静态方法，只是不能实例化），但是可以通过子类化并将子类实例化，会误导用户以为是它是为了继承而设计的。


**有一些简单的方法确保类不可被实例化。**

## 私有构造器

首先要阻止编译器自动生成默认的缺省构造器，我们要让类包含一个 **私有构造器**。

```java
// Noninstantiable utility class
public class UtilityClass {
    // Suppress default constructor for noninstantiability
    private UtilityClass() {
        throw new AssertionError();
    }

    // Remainder omitted
}
```

AssertionError 不是必须的，但能避免不小心在类内部调用构造器。

副作用是: 这个类也不能被子类化（所有构造器都必须显示或隐式调用超类构造器，这种情况下子类就没有可访问的超类构造器可调用了）。

