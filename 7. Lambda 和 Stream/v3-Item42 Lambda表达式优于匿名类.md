# v3-Item42 Lambda表达式优于匿名类



在 Java 8 中，添加了「**函数式接口**」，「**lambda表达式**」和「**方法引用**」，以便更容易地创建函数对象。 Stream API 随着其他语言的修改一同被添加进来，为处理数据元素序列提供类库支持。



---



历史原因，单一抽象方法的接口（或比较少见的抽象类）被用作「**函数类型**」，其实例被称为「函数对象」，用来表示功能或动作。自从 JDK 1.1 以来，创建函数对象的主要方法是「**匿名类**」。



```java
// Anonymous class instance as a function object - obsolete!
Collections.sort(words, new Comparator<String>() {
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});
```



匿名类适用于需要「函数对象」的经典面向对象设计模式，特别是「**策略模式**」。`Comparator`接口代表一个抽象策略 for 排序，上述的匿名类是一个具体策略 for 字符串排序。然而匿名类的冗长，使得Java中的函数式编程成为一种吸引人的前景。

在Java 8中，语言形式化了这样的概念，即使用单个抽象方法的接口是特别的，应该得到特别的对待。 这些接口现在称为「函数式接口」，并且该语言允许你使用 lambda 表达式或简称 lambda 来创建这些接口的实例。 Lambdas 在功能上与匿名类相似，但更为简洁。 下面的代码使用lambdas替换上面的匿名类。 样板不见了，行为清晰明了：

```java
// Lambda expression as function object (replaces anonymous class)
Collections.sort(words,
        		 (s1, s2) -> Integer.compare(s1.length(), s2.length()));
```

请注意，代码中不存在 lambda（`Comparator<String>`），其参数（s1和s2，都是String类型）及其返回值（int）的类型。 编译器使用称为类型推断的过程从上下文中推导出这些类型。 在某些情况下，编译器将无法确定类型，必须指定它们。 类型推断的规则很复杂：他们在JLS中占据了整个章节。 很少有程序员详细了解这些规则，但没关系。 除非它们的存在使你的程序更清晰，否则省略所有 lambda 参数的类型。 如果编译器生成一个错误，告诉你它不能推断出lambda参数的类型，那么指定它。 有时你可能不得不强制转换返回值或整个lambda表达式，但这很少见。



 如果你没有提供这些信息，编译器将无法进行类型推断，你必须在lambdas中手动指定类型，这将大大增加它们的冗余度。 举例来说，如果变量被声明为原始类型 List 而不是参数化类型`List<String>`，则上面的代码片段将不会编译。



顺便提一句，如果使用「**比较器构造方法**」代替lambda，则代码中的比较器可以变得更加简洁：

```java
Collections.sort(words, comparingInt(String::length));
```



实际上，通过利用添加到 Java 8 中的 List 接口的 sort 方法，可以使片段变得更简短：

```java
words.sort(comparingInt(String::length));
```

将 lambdas 添加到该语言中，使得使用函数对象在以前没有意义的地方非常实用。例如之前的 Operation 枚举类型。由于每个枚举都需要不同的应用程序行为，所以我们**使用了特定于常量的类主体**，并在每个枚举常量中重写了`apply`方法。



```java
// Enum type with constant-specific class bodies and data
public enum Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };
    
    private final String symbol;	// 特定于常量的数据(符号)
    Operation(String symbol) { this.symbol = symbol; }	// 构造函数
    @Override public String toString() { return symbol; }	// toString 
    public abstract double apply(double x, double y);	// 特定于常量的方法
}
```



```java

import java.util.function.DoubleBinaryOperator;

// Enum with function object fields & constant-specific behavior (Page 195)
public enum Operation {
    PLUS  ("+", (x, y) -> x + y),
    MINUS ("-", (x, y) -> x - y),
    TIMES ("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x / y);

    private final String symbol;
    private final DoubleBinaryOperator op;

    Operation(String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
    }

    @Override public String toString() { return symbol; }

    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }

    // Main method from Item 34 (Page 163)
    public static void main(String[] args) {
        double x = Double.parseDouble(args[0]);
        double y = Double.parseDouble(args[1]);
        for (Operation op : Operation.values())
            System.out.printf("%f %s %f = %f%n",
                    x, op, y, op.apply(x, y));
    }
}
```



请注意，我们使用表示枚举常量行为的 lambdas 的`DoubleBinaryOperator`接口。 这是`java.util.function`中许多预定义的函数接口之一。 它表示一个函数，它接受两个 `double` 类型参数并返回`double`类型的结果。

## 什么时候不要用 lambda 以及缺点

> 看看基于 lambda 的`Operation`枚举，你可能会认为常量特定的方法体已经失去了它们的用处，但事实并非如此。 与方法和类不同，**lambda没有名称和文档; 如果计算不是自解释的，或者超过几行，则不要将其放入lambda表达式中**。 **一行代码对于 lambda 说是理想的，三行代码是合理的最大值**。 如果违反这一规定，可能会严重损害程序的可读性。 如果一个lambda很长或很难阅读，要么找到一种方法来简化它或重构你的程序来消除它。 此外，传递给枚举构造方法的参数在静态上下文中进行评估。 因此，枚举构造方法中的lambda表达式不能访问枚举的实例成员。 如果枚举类型具有难以理解的常量特定行为，无法在几行内实现，或者需要访问实例属性或方法，那么常量特定的类主体仍然是行之有效的方法。

同样，你可能会认为匿名类在lambda时代已经过时了。 这更接近事实，但有些事情你可以用匿名类来做，而却不能用lambdas做。 Lambda仅限于函数式接口。 **如果你想创建一个抽象类的实例，你可以使用匿名类来实现，但不能使用lambda。** 同样，你可以**使用匿名类来创建具有多个抽象方法的接口实例**。 最后，**lambda不能获得对自身的引用**。 在lambda中，this 关键字引用封闭实例，这通常是你想要的。 在匿名类中，this 关键字引用匿名类实例。 *如果你需要从其内部访问函数对象，则必须使用匿名类。*

Lambdas与匿名类共享无法可靠地序列化和反序列化实现的属性。因此，**应该很少(如果有的话)序列化一个lambda(或一个匿名类实例)**。如果有一个想要进行序列化的函数对象，比如一个Comparator，那么使用一个私有静态嵌套类的实例。

综上所述，从Java 8开始，lambda 是迄今为止表示小函数对象的最佳方式。 **除非必须创建非函数式接口类型的实例，否则不要使用匿名类作为函数对象**。 另外，请记住，lambda表达式使代表小函数对象变得如此简单，以至于它为功能性编程技术打开了一扇门，这些技术在Java中以前并不实用。



