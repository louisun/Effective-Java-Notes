# Item 8: 重写equals时请遵守通用规定

虽然`Object`是一个具体的类，但它主要是为继承而设计的。它的所有非 final方法`(equals、hashCode、toString、clone和finalize)` 都有清晰的通用约定（ general contracts），因为**它们被设计为被子类重写**。任何类都有义务重写这些方法，以遵从他们的通用约定；如果不这样做，将会阻止其他依赖于约定的类(例如HashMap和HashSet)与此类一起正常工作。

本章论述何时以及如何重写Object类的非final 的方法，本章省略了finalize方法，上一章已经讨论过。

---



## 不写 equals 的情况



重写`equals`方法看起来很简单，但是有很多方式会导致重写出错，其结果可能是可怕的。

避免此问题的最简单方法是**不覆盖`equals`方法，在这种情况下，类的每个实例只与自身相等。**

如果满足以下任一下条件，则说明是正确的做法：

- **每个类的实例都是固有唯一的**。 对于像Thread这样代表活动实体而不是值的类来说，这是正确的。 Object提供的equals实现对这些类完全是正确的行为。

- **类不需要提供一个“逻辑相等（logical equality）”的测试功能**。例如`java.util.regex.Pattern`可以重写equals 方法检查两个是否代表完全相同的正则表达式Pattern实例，但是设计者并不认为客户需要或希望使用此功能。在这种情况下，从Object继承的equals实现是最合适的。

- **父类已经重写了equals方法**，则父类行为完全适合于该子类。例如，大多数Set从AbstractSet继承了equals实现、List从AbstractList继承了equals实现，Map从AbstractMap的Map继承了equals实现。

- **类是私有的或包级私有的**，可以确定它的equals方法永远不会被调用。如果你非常厌恶风险，可以重写equals方法，以确保不会被意外调用：

```java
@Override public boolean equals(Object o) {
    throw new AssertionError(); // Method is never called
}
```



## 需要重写 equals 的情况 



**如果一个类包含一个逻辑相等（ logical equality）的概念，此概念有别于对象标识（object identity），而且父类还没有重写过equals 方法。**

**这通常用在值类（ value classes）的情况。**值类只是一个表示值的类，例如 `Integer` 或 `String` 类。程序员使用`equals`方法比较值对象的引用，**期望发现它们在逻辑上是否相等，而不是引用相同的对象**。重写 equals方法不仅可以满足程序员的期望，它还支持重写过`equals` 的实例**作为Map 的键**（key），**或者 Set 里的元素**，以满足预期和期望的行为。



>  不过有一种“值类”不要覆盖 `equals` 方法，即实例受控（Item 1）的类，以确保每个值至多存在一个对象。**枚举类型**属于这个类别。 **对于这些类，逻辑相等与对象标识是一样的，所以Object的equals方法作用逻辑equals方法。**





## equals 方法实现了等价关系

当你重写equals方法时，必须遵守它的通用约定。

1. 自反性：对于任何非空引用x，`x.equals(x)`必须返回 true。
2. 对称性：对于任何非空引用x和y，如果且仅当`y.equals(x)`返回true时`x.equals(y)`必须返回true。
3. 传递性：对于任何非空引用x、y、z，如果`x.equals(y)`返回true，`y.equals(z)`返回true，则`x.equals(z)`必须返回true。
4. 一致性：对于任何非空引用x和y，如果在equals比较中使用的信息没有修改，则`x.equals(y)`的多次调用必须始终返回true或始终返回false。
5. 对于任何非空引用x，`x.equals(null)`必须返回false。



**自反性（Reflexivity）**：

第一个要求只是说一个对象必须与自身相等。 很难想象无意中违反了这个规定。 如果你违反了它，然后把类的实例添加到一个集合中，那么contains方法可能会说集合中没有包含刚添加的实例。



**对称性（Symmetry）**：

第二个要求是，任何两个对象必须在是否相等的问题上达成一致。与第一个要求不同的是，我们不难想象在无意中违反了这一要求。例如，考虑下面的类，它实现了不区分大小写的字符串。字符串被toString保存，但在equals比较中被忽略：

```java
import java.util.Objects;

public final class CaseInsensitiveString {
    private final String s;

    public CaseInsensitiveString(String s) {
        this.s = Objects.requireNonNull(s);
    }

    // Broken - violates symmetry!
    @Override
    public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString)
            return s.equalsIgnoreCase(
                    ((CaseInsensitiveString) o).s);
        if (o instanceof String)  // One-way interoperability!
            return s.equalsIgnoreCase((String) o);
        return false;
    }
    ...// Remainder omitted
}

CaseInsensitiveString cis = new CaseInsensitiveString("Polish");
String s = "polish”;

System.out.println(cis.equals(s)); // true
System.out.println(s.equals(cis)); // false
```

假设将 `CaseInsensitiveString`放到 List 里面：

```java
List<CaseInsensitiveString> list = new ArrayList<>();
list.add(cis);

list.contains(s) // 不确定会返回什么
```



`list.contains(s)` 在当前的OpenJDK实现中，它会返回false，但这只是一个实现构件。在另一个实现中，它可以很容易地返回true或抛出运行时异常。一旦违反了equals约定，就不知道其他对象在面对你的对象时会如何表现了。



```java
@Override
public boolean equals(Object o) {
    return o instanceof CaseInsensitiveString &&
            ((CaseInsensitiveString) o).s.equalsIgnoreCase(s);
}
```



## 总结



综合起来，以下是编写高质量equals方法的配方（recipe）：



1. 使用==运算符检查参数是否为该对象的引用。如果是，返回true。这只是一种性能优化，但是如果这种比较可能很昂贵的话，那就值得去做。

2. 使用`instanceof`运算符来检查参数是否具有正确的类型。 如果不是，则返回false。 通常，正确的类型是equals方法所在的那个类。 有时候，改类实现了一些接口。 如果类实现了一个接口，该接口可以改进 equals约定以允许实现接口的类进行比较，那么使用接口。 集合接口（如Set，List，Map和Map.Entry）具有此特性。

3. 参数转换为正确的类型。因为转换操作在 `instanceof` 中已经处理过，所以它肯定会成功。

4. 对于类中的每个“重要”的属性，请检查该参数属性是否与该对象对应的属性相匹配。如果所有这些测试成功，返回true，否则返回false。如果步骤2中的类型是一个接口，那么必须通过接口方法访问参数的属性;如果类型是类，则可以直接访问属性，这取决于属性的访问权限。





对于类型为非float或double的基本类型，使用`==`运算符进行比较；

对于对象引用属性，递归地调用equals方法；

对于float 基本类型的属性，使用静态`Float.compare(float, float)`方法；

对于double 基本类型的属性，使用`Double.compare(double, double)`方法。

由于存在`Float.NaN`，`-0.0f`和类似的double类型的值，所以需要对`float`和`double`属性进行特殊的处理；虽然你可以使用静态方法`Float.equals`和`Double.equals`方法对`float`和`double`基本类型的属性进行比较，这会导致每次比较时发生自动装箱，引发非常差的性能



对于数组属性，将这些准则应用于每个元素。 如果数组属性中的每个元素都很重要，请使用其中一个重载的Arrays.equals方法。

某些对象引用的属性可能合法地包含null。 为避免出现`NullPointerException`异常，请使用静态方法 `Objects.equals(Object, Object)`检查这些属性是否相等。

在下面这个简单的`PhoneNumber`类中展示了根据之前的配方构建的`equals`方法：

```java
public final class PhoneNumber {

    private final short areaCode, prefix, lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "area code");
        this.prefix = rangeCheck(prefix, 999, "prefix");
        this.lineNum = rangeCheck(lineNum, 9999, "line num");
    }

    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max)
            throw new IllegalArgumentException(arg + ": " + val);
        
        return (short) val;
    }

    @Override
    public boolean equals(Object o) { // 注意参数是 Object
        if (o == this)
            return true;
        if (!(o instanceof PhoneNumber))
            return false;

        PhoneNumber pn = (PhoneNumber) o;

        return pn.lineNum == lineNum && pn.prefix == prefix
                && pn.areaCode == areaCode;
    }

    ... // Remainder omitted
}
```



以下是一些最后提醒：

1. **当重写equals方法时，同时也要重写hashCode方法（条目 11）。**
2.  **不要让equals方法试图太聪明**。如果只是简单地测试用于相等的属性，那么要遵守equals约定并不困难。如果你在寻找相等方面过于激进，那么很容易陷入麻烦。一般来说，考虑到任何形式的别名通常是一个坏主意。例如，File类不应该试图将引用的符号链接等同于同一文件对象。幸好 File 类并没这么做。
3.  **在equal 时方法声明中，不要将参数Object替换成其他类型**。对于程序员来说，编写一个看起来像这样的equals方法并不少见，然后花上几个小时苦苦思索为什么它不能正常工作：

```java
// Broken - parameter type must be Object!
public boolean equals(MyClass o) {   
 …
}
```

问题在于这个方法并没有重写Object.equals方法，它的参数是Object类型的，这样写只是重载了 equals 方法。 即使除了正常的方法之外，提供这种“强类型”的equals方法也是不可接受的，因为它可能会导致子类中的Override注解产生误报，提供不安全的错觉。

在这里，使用Override注解会阻止你犯这个错误。这个equals方法不会编译，错误消息会告诉你到底错在哪里。





编写和测试equals(和hashCode)方法很繁琐，生的代码也很普通。替代手动编写和测试这些方法的优雅的手段是，使用谷歌 `AutoValue` 开源框架，该框架自动为你生成这些方法，只需在类上添加一个注解即可。在大多数情况下，AutoValue框架生成的方法与你自己编写的方法本质上是相同的。



很多 IDE（例如 Eclipse，NetBeans，IntelliJ IDEA 等）也有生成equals和hashCode方法的功能，但是生成的源代码比使用AutoValue框架的代码更冗长、可读性更差，不会自动跟踪类中的更改，因此需要进行测试。这就是说，使用IDE工具生成equals(和hashCode)方法通常比手动编写它们更可取，因为IDE工具不会犯粗心大意的错误，而人类则会。



总之，除非必须：在很多情况下，不要重写equals方法，从Object继承的实现完全是你想要的。 如果你确实重写了equals 方法，那么一定要比较这个类的所有重要属性，并且以保护前面equals约定里五个规定的方式去比较。



