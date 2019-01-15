# Item41. 慎用重载



「**重载**」是 `Overload`，注意和「**重写**」 `Override` 作区别

下面程序根据一个集合是 Set、List 还是其他的集合类型，带对它分类

```java
public class CollectionClassifier {
    public static String classify(Set<?> s) {
        return "Set";
    }

    public static String classify(List<?> lst) {
        return "List";
    }

    public static String classify(Collection<?> c) {
        return "Unknown Collection";
    }

    public static void main(String[] args) {
        Collection<?>[] collections = {
                new HashSet<String>(),
                new ArrayList<BigInteger>(),
                new HashMap<String, String>().values()
        };

        for (Collection<?> c : collections)
            System.out.println(classify(c));
    }
}
```



实际上，该程序只会输出 “Unknown Collection” 3 次，因为 `classify` 方法被重载（`overloaded`），要调用哪个重载方法是在「**编译时决定的**」。`for`循环的 3 次迭代，参数的编译时类型都是「`Collection<?>`」

> 对于重载方法（overloaded method）的选择是「静态的」，对于被覆盖的方法（重写方法，overridden method）的选择是「动态的」。



上述程序的意图是：期望编译器根据参数的「**运行时类型**」自动将调用分发给适当的「**重载方法**」，以此来识别参数的类型。但是**方法重载机制完全不能提供这样的功能**，一个最佳修正方案：用单个方法来替换这3个重载的 `classify` 方法，并且做一个「`instanceof`」测试：

```java
public static String classify(Collection<?> c){
    return c instanceof Set ? "Set":
    	   c instanceof List ? "List" : "Unkonwn Collection";
}
```





覆盖机制是规范，而重载机制是例外，覆盖机制满足了人们对于方法调用行为的期望，而重载机制容易让这些期望落空。如果 API 的用户不知道「对于一组给定参数，其中哪个重载方法会被调用」，那么可以回导致错误。应该「**避免胡乱地使用重载机制**」。



## 怎样才算胡乱地使用重载机制

**安全保守的策略**：永远不要导出两个「**具有相同参数数目**」的**重载方法**，除 42 条的情形之外。参数数目不同，则程序员一定能确定是哪个方法是适用的。如果一定要有相同的参数数目，就可以给两个方法**起不同的方法名。**



举例：`ObjectOutputStream` 类的 `write` 方法都有一种变形，但这种变形并不是重载 `write` 方法，而是取不同的名字用来区分，比如 `writeBoolean(boolean)`、`writeInt(int)`、`writeLong(long)`等等。这种命名模式的好处是，有可能提供相应名称的读方法，比如 `readBoolean()`、`readInt()`、`readLong()`。



**不过对于构造器 Constructor，不能选择不同的名称，因此一个类的构造器总是重载的。**很多情况下，可以选择导出静态工厂，而不是构造器（见第 1 条）。





如果对每一对重载方法，至少有 1 个对应参数在两个方法中有「根本不同」的类型，就不会混淆。如果「显然不可能把一种类型的实例转换为另一种」，那么两种类型是「根本不同」的。这种情况下，用哪个重载方法完全由「参数的运行时类型」来确定。



## 泛型、装箱导致的重载问题



Java 1.5 之前，所有基本类型都根本不同于所有引用类型，但当自动装箱出现后，就不再如此了。





```java
import java.util.*;

// What does this program print?
public class SetList {
    public static void main(String[] args) {
        Set<Integer> set = new TreeSet<>();
        List<Integer> list = new ArrayList<>();

        // -3 -2 -1 0 1 2
        for (int i = -3; i < 3; i++) {
            set.add(i);
            list.add(i);
        }
        // i：0 1 2 
        for (int i = 0; i < 3; i++) {
            // set remove 0 1 2
            set.remove(i);
            // list remove -3 -1  1
            list.remove(i);
        }
        System.out.println(set + " " + list);
    }
}

```



程序将 -3 到 2 之间的整数加到排好序的集合和列表中，然后都调用 3 次 `remove` 操作，大多数人的意图可能是去除「0，1，2」这三个值，留下 「-3，-2，-1」这三个值，但是，`list` 打印出来的是「-2, 0, 2」。这种行为称之为混乱，已经是保守的说法。也就是说 `ArrayList` 的 remove 是继承 `AbstractList` 中的 `remove(index)`，实际去除的是下标，注意是迭代地 remove，下标对应的元素是变化的。

> 其实就如 `ArrayList` 这种 `remove` 有不同参数，一个是 int 删除下标，另一种是 Object 删除对象，当引入「泛型」后，`ArrayList<Integer>` 的 `remove(Object)` 就是 `remove(Integer)`，又当引入「装箱」后，int 可能会被装箱，这两种参数类型就不再根本不同了，这样调用哪个函数就很不明确了，因此破坏了 List 接口。

`set` 就没有 `remove(int)`删除下标这种操作，默认就是 `remove(Object)`，`int` 也会被自动装箱为 `Integer`。



因此引入泛型和自动装箱后，要更谨慎地使用重载，还好 Java 的类库中上面这个是特例，其他没有 API 被破坏。



另外，「**数组类型**」和 Object 之外的类都截然不同。数组类型和 `Serializable`与 `Cloneable`之外的接口也截然不同。如果两个类都不是对方的后代，这两个独特的类就是不相关的。任何对象都不可能两个不相关的类的实例，因此不相关的类是根本不同的。





## 总结

简而言之，"能够重载方法"并不意味着就"应该重载方法"。一般情况下，对于多个具有相同参数数目的方法来说，应该尽量避免重载方法。在某些情况下，特别是涉及构造器的时侯，要遵循这条建议也许是不可能的。在这种情况下，至少应该避免这样的情形：同一组参数只需经过类型转换就可以被传递给不同的重载方法。如果不能避免这种情形，例如，因为正在改造一个现有的类以实现新的接口，就应该保证：当传递同样的参数时，所有重载方法的行为必须一致。如果不能做到这一点，程序员就很难有效地使用被重载的方法或者构造器，他们就不能理解它为什么不能正常地工作。







