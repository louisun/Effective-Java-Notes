# Item28. 使用限定通配符来增加API的灵活性

[TOC]



如条目 25 所述，参数化类型是不变的。换句话说，对于任何两个不同类型的`Type1`和`Type2`，`List <Type1>`既不是`List <Type2>`子类型也不是其父类型。尽管`List<String>`不是`List<Object>`的子类型是违反直觉的，但它确实是有道理的。 可以将任何对象放入`List<Object>`中，但是只能将字符串放入`List<String>`中。 由于`List<String>`不能做`List<Object>`所能做的所有事情，所以它不是一个子类型（里氏替代原则）。

## 1. 不使用通配符



有时候，我们需要的灵活性比不可变类型所能提供的更多。 考虑条目 26 中的`Stack`类。

```java
public class Stack<E> {
    public Stack();
    public void push(E e);
    public E pop();
    public boolean isEmpty();
}
```



```java
// pushAll method without wildcard type - deficient!
public void pushAll(Iterable<E> src) {
    for (E e : src)
        push(e);
}
```

可以正确编译，但也可能出问题。元素类型匹配是运行正常的，  但若一个`Stack<Number>`，并调用`push(intVal)`，其中`intVal`的类型是`Integer`。 这是因为`Integer`是`Number`的子类型。 从逻辑上看，这似乎也应该起作用：

```java
Stack<Number> numberStack = new Stack<>();
Iterable<Integer> integers = ... ;
numberStack.pushAll(integers);
```

但会得到这个错误消息，**因为参数化类型是不变的：**

```java
StackTest.java:7: error: incompatible types: Iterable<Integer>
cannot be converted to Iterable<Number>
        numberStack.pushAll(integers);
                            ^
```

## 2. 限定通配符

`Iterable <？extends E>`。 （关键字`extends`有点误导，实际是表示都是 `E` 的子类型）：

```java
// 使用限定通配符的参数：src 是一个生产者(可以 get，提供给外界参数)
// 注意这个 E 是 Stack 的 E
public void pushAll(Iterable<? extends E> src) {
    for (E e : src)
        push(e);
}
Stack<Number> numberStack = new Stack<>();
Iterable<Integer> integers = ... ;
// 可以将 E: Number 的子类 Integer 作为类型参数
numberStack.pushAll(integers);	
```



```java
// 使用限定通配符的参数：dst 是一个消费者(可以 set，使用外界提供的参数)
// 注意这个 E 是 Stack 的 E
public void popAll(Collection<? super E> dst) {
    while (!isEmpty())
        dst.add(pop());
}

Stack<Number> numberStack = new Stack<Number>();
Collection<Object> objects;
// 可以将 Number 的父类 Object 作为类型参数
numberStack.popAll(objects);	
```



## 3. 选择限定通配符类型 PECS

**如果一个输入参数既是一个生产者又是一个消费者，那么通配符类型对你没有好处。**

**你需要一个精确的类型匹配，这就是没有任何通配符的情况。**

有一个助记符来帮助你记住使用哪种通配符类型：

> **PECS代表： producer-extends，consumer-super。**
>
> 生产者（从里面get、或者是提供外界使用该参数） `extends`，消费者 (set到里面，或从使用外接的提供的参数) `super`



记住这个助记符之后，让我们来看看本章中以前项目的一些方法和构造方法声明。 `Chooser`类构造方法有这样的声明：

```java
public Chooser(Collection<T> choices)
```

这个构造方法只使用集合选择来生产类型`T`的值（并将它们存储起来以备后用），所以它的声明应该使用一个`extends T`的通配符类型。下面是得到的构造方法声明：

```java
// Wildcard type for parameter that serves as an T producer
public Chooser(Collection<? extends T> choices)
```

这种改变在实践中会有什么不同吗？ 是的，会有不同。 假你有一个`List <Integer>`，并且想把它传递给`Chooser<Number>`的构造方法。 这不会与原始声明一起编译，但是它只会将限定通配符类型添加到声明中。



### 3.1 union 两个 Set



现在看看的`union`方法。下是声明：

```java
public static <E> Set<E> union(Set<E> s1, Set<E> s2)
```

两个参数`s1`和`s2`都是`E`的生产者，所以 **PECS** 助记符告诉我们该声明应该如下：

```java
public static <E> Set<E> union(Set<? extends E> s1,  Set<? extends E> s2)
```

请注意，返回类型仍然是`Set<E>`。 不要使用限定通配符类型作为返回类型。除了会为用户提供额外的灵活性，还强制他们在客户端代码中使用通配符类型。 通过修改后的声明，此代码将清晰地编译：

```java
Set<Integer>  integers =  Set.of(1, 3, 5);
Set<Double>   doubles  =  Set.of(2.0, 4.0, 6.0);

// union 可以接受两种：Set<Integer>， Set<Double> 做为参数
Set<Number>   numbers  =  union(integers, doubles);

// Java 8 之前的推断规则不智能，只能
Set<Number>	  numbers = <Number>union(integers, double)
```

如果使用得当，类的用户几乎不会看到通配符类型。 他们使方法接受他们应该接受的参数，拒绝他们应该拒绝的参数。 **如果一个类的用户必须考虑通配符类型，那么它的API可能有问题。**



### 3.2 max 一个 List 中的元素  （复杂）

原始声明：`T` 是一个*可比较类型*的子类型

```java
public static <T extends Comparable<T>> T max(List<T> list)
```

使用通配符类型的修改后：

```java
public static <T extends Comparable<? super T>> T max(List<? extends T> list)
```

为了从原来到修改后的声明，我们两次应用了**PECS**。首先直接的应用是参数列表。 它生成`T`实例，所以将类型从`List<T>`更改为`List<? extends T>`。

 棘手的是类型参数`T`。这是我们第一次看到**通配符应用于类型参数**。 最初，`T`被指定为继承`Comparable <T>`，但`Comparable`的`T`消费`T`实例（**并生成指示顺序关系的整数**）。 因此，参数化类型`Comparable <T>`被替换为限定通配符类型`Comparable<? super T>`。 `Comparable`实例总是消费者，所以通常应该**使用`Comparable<? super T>`优于`Comparable <T>`**。 `Comparator`也是如此。因此，通常应该**使用`Comparator<? super T>`优于`Comparator<T>`**。

> **思考：Comparable 为什么是一个消费者？哪里消费 T 了？**想想 `compareTo(T t)` 方法
>
> 是**使用了外部的 T 参数**，返回一个整数结果，并没有生产或提供 T 类型，外部的 t 变量才是生产者？



修改后的`max`声明可能是本书中最复杂的方法声明。这是一个列表的简单例子，它被原始声明排除，但在被修改后的版本里是允许的：

```java
List<ScheduledFuture<?>> scheduledFutures = ... ;

// 原始类型
public static <T extends Comparable<T>> T max(List<T> list)
// ScheduledFuture 并不实现 Comparable<ScheduledFuture>，不满足限定参数要求
// ScheduledFuture 继承的是 Comparable<Delayed>，如何变成复合参数的要求呢？
    
public static <T extends Comparable<super T>> T max(List<T> list)
```

无法将原始方法声明应用于此列表的原因是`ScheduledFuture`不实现`Comparable<ScheduledFuture>`。 相反，它是`Delayed`的子接口，它继承了`Comparable<Delayed>`。 换句话说，一个`ScheduledFuture`实例不仅仅和其他的`ScheduledFuture`实例相比较： 它可以与任何`Delayed`实例比较，并且足以导致原始的声明拒绝它。 更普遍地说，通配符要求来支持没有直接实现`Comparable`（或`Comparator`）的类型，但继承了一个类型。



## 4. 通配符`<?>`捕获：辅助函数 `<T>`

 类型参数和通配符之间具有双重性，许多方法可以用其中一个或另一个声明。 例如，下面是两个可能的声明，用于交换列表中两个索引项目的静态方法。 第一个使用**无限制类型参数**，第二个使用无**限制通配符**：

```java
// Two possible declarations for the swap method
public static <E> void swap(List<E> list, int i, int j);
public static void swap(List<?> list, int i, int j);
```

这两个声明中的哪一个更可取，为什么？ 在公共 API 中，第二个更好，因为它更简单。 你传入一个列表（任何列表），该方法交换索引的元素。 没有类型参数需要担心。 通常，**如果类型参数在方法声明中只出现一次，请将其替换为通配符**。 如果它是一个无限制的类型参数，请将其替换为无限制的通配符; 如果它是一个限定类型参数，则用限定通配符替换它。

**第二个`swap`方法声明有一个问题。 这个简单的实现不会编译：**

```java
public static void swap(List<?> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));
}
```

试图编译它会产生这个不太有用的错误信息：

```java
Swap.java:5: error: incompatible types: Object cannot be
converted to CAP#1
        list.set(i, list.set(j, list.get(i)));
                                        ^
  where CAP#1 is a fresh type-variable:
    CAP#1 extends Object from capture of ?
```

看起来我们不能把一个元素放回到我们刚刚拿出来的列表中。 **问题是列表的类型是`List<?>`，并且不能将除`null`外的任何值放入`List<?>`中。**



 **幸运的是，有一种方法可以在不使用不安全的转换或原始类型的情况下实现此方法。 这个想法是写一个私有辅助方法来捕捉通配符类型。** 辅助方法必须是泛型方法才能捕获类型。 以下是它的定义：

```java
public static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
}
// 辅助函数
private static <E> void swapHelper(List<E> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));
}
```

`swapHelper`方法知道该列表是一个`List<E>`。 因此，它知道从这个列表中取出的任何值均为E类型，并且知道将`E`类型的任何值放进列表都是安全的。`swap`这个有些费解的实现编译起来却是正确无误的。它允许我们导出`swap`这个比较好的基于通配符的声明，同时在内部利用更加复杂的泛型方法。`swap`方法的客户端不一定要面对更加复杂的`swapHelper`声明，但是它们的确从中受益。



## 总结



总而言之，在API中使用通配符类型虽然比较需要技巧，但是使API变得灵活得多。如果编写的是将被广泛使用的类库，则一定要适当地利用通配符类型。记住基本的原则：producerextends，consumer - super（PECS）。还要记住所有的comparable和comparator都是消费者。






