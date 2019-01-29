# Item25. 列表优先于数组



## 区别一

数组是**协变**的（convariant），即如果 Sub 是 Super 的子类型，那么 `Sub[]` 就是 `Super[]` 的子类型。



泛型是不可变的（invariant），不同的类型 Type1 和 Type2，`List<Type1>` 和 `List<Type2>` 的关系不是子类也不是父类。



因此下列代码是编译时是合法的，但运行时不合法：

```java
Object[] objectArray = new Long[1]; // 这两种数组是父类和子类的关系，语法上没问题
objectArray[0] = "I don't fit in"; // 抛出 ArrayStoreException
```



下列是不合法的，在编译时候就发现错误：

```java
List<Object> ol = new ArrayList<Long>;	// 这两种类型不是父类和子类的关系，不能赋值
ol.add("I don't fit in");
```



## 区别二

**数组是具体化的（reified）**，在运行时候才知道并检查元素类型约束，比如上面企图将 String 保存到 Long 数组中。



泛型是通过擦除（erasure）来实现的。**泛型只在编译时强化它们的类型信息，在运行时丢弃（或擦除）它们的元素类型信息**。擦除就是可以使泛型可以与没有使用泛型的代码随意互用。



因此泛型与数组不能很好地混合使用，例如创建泛型、参数化类型或类型参数的数组都是非法的。



```java
// 都非法，出现 generic array creation 错误
new List<E>[]
new List<String>[]
new E[]
```



为什么？因为混用后不是类型安全的，可能会出现异常，违背了泛型系统提供的基本保证。用反证法，假设它是合法的：

![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2018-12-03-23-38-31_r1.png)

第五行会出现 `ClassCastException` 异常，为了防止出现这种情况，第一	行就会产生编译错误。





像 `E`、`List<E>`、`List<String>` **都是不可具体化类型。**就是指运行时表示法包含的信息比编译时表示法包含的信息更少的类型。唯一可具体化的参数化类型是『无限制通配符』类型，如 `List<?>`和 `Map<?,?>`，虽然不常用，但是创建无限制通配符类型的数组是合法的。





泛型一般不可能返回它的元素类型数组。这也意味着结合使用可变参数（varargs）方法和泛型时会出现令人费解的警告。除了 `@SupressWarnings`，或避免混用泛型与可变参数，别无他法。



## 解决方法：`List<E>`



当得到泛型数组创建错误时，最好是优先使用集合类型 `List<E>` 而不是数组类型 `E[]`。 这样可能会牺牲一些简洁性或性能，但作为交换，你会获得更好的类型安全性和互用性。



例如，假设你想用带有集合的构造方法来编写一个Chooser类，并且有个方法返回随机选择的集合的一个元素。 根据传递给构造方法的集合，可以使用选择器作为游戏模具，魔术8球或数据源进行蒙特卡罗模拟。 这是一个没有泛型的简单实现：



```java
// Chooser - a class badly in need of generics!
public class Chooser {
    private final Object[] choiceArray;

    public Chooser(Collection choices) {
        choiceArray = choices.toArray();
    }

    public Object choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceArray[rnd.nextInt(choiceArray.length)];
    }
}
```



要使用这个类，每次调用方法时，都必须将Object的`choose`方法的**返回值转强制换为所需的类型**，如果类型错误，则转换在运行时失败。 我们先试图修改`Chooser`类，使其成为泛型的。



```java
// A first cut at making Chooser generic - won't compile
public class Chooser<T> {
    private final T[] choiceArray;

    public Chooser(Collection<T> choices) {
        choiceArray = choices.toArray(); // toArray 返回的是一个 Object[]
    }

    public Object choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceArray[rnd.nextInt(choiceArray.length)];
    }
}
```



如果你尝试编译这个类，会得到这个错误信息：

```java
Chooser.java:9: error: incompatible types: Object[] cannot be
converted to T[]
        choiceArray = choices.toArray();
                                     ^
  where T is a type-variable:
    T extends Object declared in class Chooser
```



没什么大不了的，将 Object 数组转换为 T 数组：

```java
choiceArray = (T[]) choices.toArray();
```



**编译器告诉你在运行时不能保证强制转换的安全性，因为程序不会知道 T 代表什么类型**——记住，元素类型信息在运行时会被泛型删除。 该程序可以正常工作吗？ 是的，**但编译器不能证明这一点。 你可以证明这一点，在注释中提出证据，并用注解来抑制警告，但最好是消除警告的原因**。



要消除未经检查的强制转换警告，请使用列表而不是数组。 下面是另一个版本的`Chooser`类，编译时没有错误或警告：



```java
// List-based Chooser - typesafe
public class Chooser<T> {
    private final List<T> choiceList; // 变成 List 了，不在是数组
    
    public Chooser(Collection<T> choices) {
        choiceList = new ArrayList<>(choices);
    }
    
    public T choose() {
        Random rnd = ThreadLocalRandom.current();
        return choiceList.get(rnd.nextInt(choiceList.size()));
    }
}

```



这个版本有些冗长，也许运行比较慢，但是值得一提的是，在运行时不会得到`ClassCastException`异常。



## 总结



总之，数组和泛型具有非常不同的类型规则。 数组是协变和具体化的；泛型是不变的，类型擦除的。 因此，数组提供运行时类型的安全性，但不提供编译时类型的安全性，反之亦然。 一般来说，数组和泛型不能很好地混合工作。 如果你发现把它们混合在一起，得到编译时错误或者警告，你的第一反应应该是用列表来替换数组。

