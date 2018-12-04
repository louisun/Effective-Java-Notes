# Item27. 优先考虑泛型方法



如类可以从泛型中收益，方法也一样。**静态工具方法**尤其适合泛型化。`Collections` 中的所有“算法”方法（例如`binarySearch`和`sort`）都泛型化了。



```java
// Uses raw types - unacceptable! [Item 26]
public static Set union(Set s1, Set s2) {
    Set result = new HashSet(s1);
    result.addAll(s2);
    return result;
}
```



这个方法可以编译，但是有两条警告：

```java

Union.java:5: warning: [unchecked] unchecked call to

HashSet(Collection<? extends E>) as a member of raw type HashSet

        Set result = new HashSet(s1);

                     ^

Union.java:6: warning: [unchecked] unchecked call to

addAll(Collection<? extends E>) as a member of raw type Set

        result.addAll(s2);

                     ^
```



为了修正这些警告，**使方法变成是类型安全的**，要将方法声明修改为**声明一个类型参数**，表示这三个集合的元素类型（两个参数和一个返回值），并在方法中使用类型参数。声明类型参数的类型参数列表，处在方法的修饰符及其返回类型之间。在这个示例中，类型参数列表为`<E>`，返回类型为`Set<E>`。类型参数的命名惯例与泛型方法以及泛型的相同：



```java
// Generic method
public static <E> Set<E> union(Set<E> s1, Set<E> s2) {
    Set<E> result = new HashSet<>(s1);
    result.addAll(s2);
    return result;
}
```



至少对于见到那的泛型方法而言，就是这么回事了。现在该方法编译时不会产生任何警告，并提供了类型安全性，也更容易使用。以下是一个执行该方法的简单程序。程序中不包含转换，编译时不会有错误或者警告：



```java
// Simple program to exercise generic method
public static void main(String[] args) {
    Set<String> guys = Set.of("Tom", "Dick", "Harry");
    Set<String> stooges = Set.of("Larry", "Moe", "Curly");
    Set<String> aflCio = union(guys, stooges);
    System.out.println(aflCio);
}
// Moe, Tom, Harry, Larry, Curly, Dick 
```



`union` 方法的**局限性**在于，三个集合的**类型（两个输入参数和一个返回值）必须全部相同。利用有限制的通配符类型（bounded wildcard type），可以使这个方法变得更加灵活。**



**泛型方法的一个显著特性是，无需明确指定类型参数的值，不像调用泛型构造器的时候是必须指定的。**编译器通过检查方法参数的类型来计算类型参数的值。对于上述的程序而言，编译器发现`union`的两个参数都是`Set<String>类`型，因此知道类型参数`E`必须为`String`。这个过程称作**类型推导**（type inference）。



如第 1 条所述，可以利用泛型方法调用所提供的类型推导，使创建参数化类型实例的过程变得更加轻松。提醒一下：在调用泛型构造器的时候，要明确传递类型参数的值可能有点麻烦。类**型参数出现在了变量声明的左右两边，显得有些冗余：**

```java
// Parameterized type instance creation with constructor
Map<String , List<String>> anagrams = new HashMap<String , List<String>>();
```



JDK 7之前的方法是用**泛型静态工厂方法**：

```java
public static <K,V> HashMap<K,V> newHashMap(){
    return new HashMap<K,V>()
}

Map<String, List<String>> anagrams = newHashMap();
```



JDK 7 采用了泛型目标类型推断，因此只要：

```java
Map<String , List<String>> anagrams = new HashMap<>();
```



## 泛型单例（不可变、又适用于不同类型）



有时，**会需要创建不可变、又适用于不同类型的对象**。由于泛型是通过擦除实现的，可以给所有必须的类型参数使用单个对象，但是需要编写一个**静态工厂方法**，重复的给每个必要的类型参数分发对象。这种模式最常用于**函数对象**，如`Collections.reverseOrder`，但也适用于像`Collections.emptySet`这样的集合。



假设有一个接口，描述了一个方法，该方法接受和返回某个类型T的值：



```java
public interface UnaryFunction<T>{
    // apply 方法适用于不同类型
    T apply(T args);
}
```



### 恒等函数

再假设要提供一个**恒等函数**（identity function）。如果在每次需要的时候都重新创建一个，这样会很浪费，因为它是无状态的（stateless）。**如果泛型被具体化了，每个类型都需要一个恒等函数，但是他们被擦除以后，就只需一个泛型单例。**请看以下示例：



```java
// Generic singleton factory pattern
private static UnaryOperator<Object> IDENTITY_FN = new UnaryOperator<Object>{
    public Object apply(Object arg){ return arg;}
}
// 可用 lambda 表达式代替上面的写法
private static UnaryOperator<Object> IDENTITY_FN = (t) -> t;


@SuppressWarnings("unchecked")
public static <T> UnaryOperator<T> identityFunction() {
    return (UnaryOperator<T>) IDENTITY_FN; // 返回这个泛型单例
}
```



`IDENTITY_FUNCTION`转换成（`UnaryFunction<T>`），产生了一条未受检的转换警告，因为`UnaryFunction<Object>`对于每个`T`来说并非都是个`UnaryFunction<T>`。



**但是恒等函数很特殊：它返回未被修改的参数**，因此我们知道无论`T`的值是什么，用它作为`UnaryFunction<T>`都是类型安全的。**因此，我们可以放心的禁止由这个转换所产生的未受检转换警告。一旦禁止，代码在编译时就不会出现任何错误或者警告。**



以下是一个范例程序，**利用泛型单例作为`UnaryFunction<String>`和`UnaryFunction<Number>`。**像往常一样，他不包含转换，编译时没有出现错误或者警告：



```java
// Sample program to exercise generic singleton
public static void main(String[] args) {
    
    // UnaryOperator<String>
    String[] strings = { "jute", "hemp", "nylon" };   
    UnaryOperator<String> sameString = identityFunction(); // 可推断 T 为 String
    for (String s : strings)
        System.out.println(sameString.apply(s));
    // UnaryOperator<Number>
    Number[] numbers = { 1, 2.0, 3L };  
    UnaryOperator<Number> sameNumber = identityFunction(); // 可推断 T 为 Number
    for (Number n : numbers)
        System.out.println(sameNumber.apply(n));

}
```



虽然相对少见，但是通过某个**包含该类型参数本身的表达式**来限制类型参数是允许的，这就是**递归类型限制**（recursive type bound）。递归类型限制最普遍的用途与`Comparable`接口有关，它定义类型的自然顺序：



```java
public interface Comparable<T> {
    int compareTo(T o);
}
```



类型参数`T`定义了实现`Comparable<T>`的类型的元素可以比较的类型。 在实际中，几乎所有类型都只能与自己类型的元素进行比较。 所以，例如，String 类实现了`Comparable<String>`，Integer 类实现了`Comparable<Integer>`等等。



许多方法采用实现`Comparable`的元素的集合来对其进行排序，在其中进行搜索，计算其最小值或最大值等。 要做到这一点，要求集合中的每一个元素都可以与其中的每一个元素相比，换言之，这个元素是可以相互比较的。 以下是如何表达这一约束



```java
// Using a recursive type bound to express mutual comparability
public static <E extends Comparable<E>> E max(Collection<E> c);
```



限定的类型`<E extends Comparable <E>>`可以理解为“**任何可以与自己比较的类型 E**”，这或多或少精确地对应于相互可比性的概念。

这里有一个与前面的声明相匹配的方法。它根据其元素的自然顺序来计算集合中的最大值，并编译没有错误或警告：



```java
// Returns max value in a collection - uses recursive type bound

public static <E extends Comparable<E>> E max(Collection<E> c) {
    if (c.isEmpty())
        throw new IllegalArgumentException("Empty collection");
    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
            result = Objects.requireNonNull(e);
    return result;
}
```



请注意，如果列表为空，则此方法将引发`IllegalArgumentException`异常。 更好的选择是返回一个`Optional<E>`。



递归类型限制可能比这个要复杂得多，但幸运的是，这种情况并不经常发生。如果你理解了这种习惯用法及其通配符变量，就能够处理在实践中遇到的许多递归类型限制了。



## 总结

总而言之，泛型方法就像泛型一样，使用起来比要求客户端转换输入参数并返回值的方法来得更加安全，也更加容易。就像类型一样，你应该确保新方法可以不用转换就能使用，这通常意味着要将它们泛型化。并且就像类型一样，还应该将现有的方法泛型化，使新用户使用起来更加轻松，且不会破坏现有的客户端。