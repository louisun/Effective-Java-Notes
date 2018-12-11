# Item34. 用接口模拟可伸缩的枚举



## 用接口扩展枚举（enum inplements sth）



可以让一个枚举类型去扩展另一个枚举类型，但枚举的可伸缩性（extensible）最后证明基本上不是什么好点子。不过对于可伸缩的枚举类型，至少有一种具有说服力的用例，这就是**操作码**（ `operation code`），也称作 `opcode`。操作码是指这样的枚举类型：它的元素表示在某种机器上的那些操作，例如第30条中的 `Operation`类型，它表示一个简单的计算器中的某些函数。有时候，要尽可能地让 API 的用户提供它们自己的操作，这样可以有效地扩展`API`所提供的操作集。



幸运的是，有一种很好的方法可以利用枚举类型来实现这种效果。由**于枚举类型可以通过给操作码类型和（属于接口的标准实现的）枚举定义接口，来实现任意接口**，基本的想法就是利用这一事实。例如，以下是第30条中的 `Operation`类型的扩展版本（之前没有用接口，是用了个抽象方法）：



```java
// Emulated extensible enum using an interface
public interface Operation {
    double apply(double x, double y);
}

public enum BasicOperation implements Operation {
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
    private final String symbol;


    BasicOperation(String symbol) {
        this.symbol = symbol;
    }

    @Override public String toString() {
        return symbol;
    }
}
```



**虽然枚举类型（`BasicOperation`）不可扩展，但接口类型（`Operation`）是可以扩展的**，并且它是用于表示API中的操作的接口类型。 你可以定义另一个实现此接口的枚举类型，并使用此新类型的实例来代替基本类型。 例如，假设想要定义前面所示的操作类型的扩展，包括指数运算和余数运算。 你所要做的就是编写一个实现`Operation`接口的枚举类型：



```java
// 模拟扩展 enum，实际是扩展接口
public enum ExtendedOperation implements Operation {
    EXP("^") {
        public double apply(double x, double y) {
            return Math.pow(x, y);
        }
    },
    
    REMAINDER("%") {
        public double apply(double x, double y) {
            return x % y;
        }
    };

    private final String symbol;

    ExtendedOperation(String symbol) {
        this.symbol = symbol;
    }

    @Override public String toString() {
        return symbol;
    }
}
```



只要API编写为接口类型（`Operation`），而不是实现（`BasicOperation`），现在就可以在任何可以使用基本操作的地方使用新操作。请注意，不必在枚举中声明`apply`抽象方法，就像您在具有实例特定方法实现的非扩展枚举中所做的那样。 这是因为抽象方法（`apply`）是接口（`Operation`）的成员。



## 传入整个扩展的枚举类型

不仅可以在任何需要“基本枚举”的地方传递“扩展枚举”的单个实例，而且还可以传入整个扩展枚举类型，并使用其元素。 例如，这里是一个测试程序版本，它执行之前定义的所有扩展操作：

```java
public static void main(String[] args) {
    double x = Double.parseDouble(args[0]);
    double y = Double.parseDouble(args[1]);
    test(ExtendedOperation.class, x, y);
}

private static <T extends Enum<T> & Operation> void test(Class<T> opEnumType, 
                                                         double x, 
                                                         double y) 
{
    for (Operation op : opEnumType.getEnumConstants())
        System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
}
```

注意，扩展的操作类型的类字面文字（`ExtendedOperation.class`）从`main`方法里传递给了`test`方法，用来描述扩展操作的集合。这个类的字面文字用作限定的类型令牌（条目 29）。`opEnumType`参数中复杂的声明（`<T extends Enum<T> & Operation> Class<T>`）**确保了Class对象既是枚举又是`Operation`的子类**，这正是遍历元素和执行每个元素相关联的操作时所需要的。





第二种方式是传递一个`Collection<? extends Operation>`，**这是一个限定通配符类型（条目 27），而不是传递了一个class对象：**



> Enum 也是一种 Collection

```java
public static void main(String[] args) {
    double x = Double.parseDouble(args[0]);
    double y = Double.parseDouble(args[1]);
    test(Arrays.asList(ExtendedOperation.values()), x, y);
}

private static void test(Collection<? extends Operation> opSet,
        				double x, double y) {
    for (Operation op : opSet)
        System.out.printf("%f %s %f = %f%n",x, op, y, op.apply(x, y));
}
```

生成的代码稍微不那么复杂，`test`方法灵活一点：它允许调用者将多个实现类型的操作组合在一起。另一方面，也放弃了在指定操作上使用`EnumSet`(条目32)和`EnumMap`(条目33)的能力。

上面的两个程序在运行命令行输入参数4和2时生成以下输出：

```
4.000000 ^ 2.000000 = 16.000000
4.000000 % 2.000000 = 0.000000
```



## 用接口扩展枚举的缺点

用接口模拟可伸缩枚举有个小小的不足，**即无法将实现从一个枚举类型继承到另一个枚举类型。**在上述 `Operation`的示例中，保存和获取与某项操作相关联的符号的逻辑代码，可以复制到 `Basicoperation`和 `Extendedoperation`中。在这个例子中是可以的，因为复制的代码非常少如果共享功能比较多，则可以将它封装在一个辅助类或者静态辅助方法中，来避免代码的复制工作。

该条目中描述的模式在Java类库中有所使用。例如，`java.nio.file.LinkOption`枚举类型实现了`CopyOption`和`OpenOption`接口。



## 总结



总之，**虽然不能编写可扩展的枚举类型，但是你可以编写一个接口来配合实现接口的基本的枚举类型，来对它进行模拟**。这允许客户端编写自己的枚举（或其它类型）来实现接口。如果API是根据接口编写的，那么在任何使用基本枚举类型实例的地方，都可以使用这些枚举类型实例。