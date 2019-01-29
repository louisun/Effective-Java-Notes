# v3-Item44 优先使用标准的函数式接口

有了 Lambda 表达式——写 API 的最佳实践已经改变了，比如之前的「模板方法模式」：子类覆盖父类方法以特殊化其行为，变得不再有吸引力。更现代的方式是，提供一个静态工厂或构造器，其接受一个「**函数对象**」来达到类似的效果。可以写更多以「**函数对象**」为参数的构造器和方法，不过要注意选择**正确的函数式参数类型**。

考虑 `LinkedhashMap`，可以覆盖其受保护的 `removeEldestEntry` 来把这个类当做缓存用，`removeEldestEntry` 在每次 put 新的 key 到 map 的时候被调用，当这个方法返回 `true`，map 删除最早的 entry，这个最早的 entry 被当作参数传入 `removeEldestEntry` 方法中。比如下面子类覆盖了这个方法：

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
   return size() > 100;
}
```

即当 `size` 大于 100 就删除最早的条目，即保留 100 个最近的 entry。

其实可以用 Lambda 做的更好，如果 `LinkedhashMap` 在 Java 8 以后编写，就会有一个以「函数对象」为参数的静态工厂或构造器，看看 `removeEldestEntry` 的声明，你可能会认为函数对象应该接受 `Map.Entry<K,V>`为参数，返回一个 `boolean`，但实际上并不完全如此。`removeEldestEntry` 方法调用 `size()` 来获得 map 中的条目数，它之所以成功是因为 `removeEldestEntry` 本身也是 map 的一个实例方法。而传入的函数对象并非一个实例方法，而且不能捕获 map，因为 map实例在工厂方法或构造器被调用时都不存在。因此，map 必须给「函数对象」传递它自身，函数对象把 map 以及最久的条目作为输入参数。如果要声明这样一个功能接口，应该是这样的：

```java
// Unnecessary functional interface; use a standard one instead.
@FunctionalInterface interface EldestEntryRemovalFunction<K,V>{
    boolean remove(Map<K,V> map, Map.Entry<K,V> eldest);
}
```



这个接口能正常工作，但不应该使用它。因为你不需要为了这个目的声明接口。`java.util.function` 包提供了大量「**标准函数接口 standard function interfaces**」供你使用。



如果其中一个「标准函数接口」能做这个工作，就应该优先使用它，而不是声明一个专门构建的函数接口。



## 好处



这将使你的 API 更易学，通过减少不必要的概念，提供交互性的好处，因为很多「标准函数结果」提供了方便的默认方法。比如 `Predicated`（谓词）接口，提供了组合判断的方法。在上面 `LinkedHashMap` 这个例子中，就应该优先使用标准函数接口 `BiPredicate<Map<K,V>, Map.Entry<K,V>>` ，而不是自定义的 `EldestEntryRemovalFunction` 函数接口，



## 6️六类基本函数接口

在`java.util.Function`包中一共有 43 个接口，不可能都记住，但只要记住 6 种「**基本接口**」，就可以推断出其余你需要的。「**基本接口**」操作于对象的引用类型。

1. `Operator`（操作者）接口表示方法的**「返回类型」**和**「参数类型」**相同。
2. `Predicate`（谓词）接口表示其方法**「接受一个参数」**并**「返回一个布尔值」**。
3. `Function`（函数）接口表示方法「**参数类型**」和「**返回类型**」不同。
4. `Supplier`（生产者）接口表示一个「**不接受参数**」，「**返回一个值**」的方法。
5. `Consumer`（消费者）接口表示该方法「**接受一个参数**」而「**不返回任何东西**」。





![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-02-20-44-16_r58.png)



## 衍生版本的函数接口

在处理基本数据类型 `int`、`long`、`double`的时候，6 种基本函数接口的每种都有 3 种变体。它们的名字是通过在**基本接口前加一个基本类型**而得到的。 因此，例如，一个接受 `int` 的`Predicate`是一个`IntPredicate`，而一个接受两个`long`值并返回一个`long`的二元运算符是一个`LongBinaryOperator`。 除`Function`接口变体通过返回类型进行了参数化，其他变体类型都没有参数化。 例如，`LongFunction<int[]>`使用`long`类型作为参数并返回了`int []`类型。



`Function`接口还有九个额外的变体，当「**结果类型为基本类型**」时使用。 参数和返回类型总是不同，因为从类型到它自身的函数是`UnaryOperator`。 如果源类型和结果类型都是基本类型，则使用带有`SrcToResult`的前缀`Functinon`，例如`LongToIntFunction`(六个变体)。如果源是一个基本类型，返回结果是一个对象引用，那么带有`<Src>ToObj`的前缀`Function`，例如`DoubleToObjFunction` (三种变体)。



还有其中 3 种基本函数接口的「**双参数**」版本：`BiPredicate<T，U>`，`BiFunction <T，U，R>`和`BiConsumer <T，U>`。 也有返回三种相关基本类型的`BiFunction`变体：`ToIntBiFunction <T，U>`，`ToLongBiFunction <T，U>`和`ToDoubleBiFunction <T，U>`。`Consumer`有两个变量，它们带有一个对象引用和一个基本类型：`ObjDoubleConsumer <T>`，`ObjIntConsumer <T>`和`ObjLongConsumer <T>`。 总共有九个两个参数版本的基本接口。



最后，还有一个`BooleanSupplier`接口，它是`Supplier`的一个变体，它「**返回布尔值**」。 这是任何标准函数式接口名称中唯一明确提及的布尔类型，但布尔返回值通过`Predicate`及其四种变体形式支持。 前面段落中介绍的`BooleanSupplier`接口和42个接口占所有四十三个标准功能接口。 无可否认，这是非常难以接受的，并且不是非常正交的。 另一方面，你所需要的大部分功能接口都是为你写的，而且它们的名字是频繁用到的，所以在你需要的时候不应该有太多的麻烦。



大多数标准函数接口仅提供「**基本类型**」的支持。 **不要试图使用基本函数接口 with 装箱的基本类型**。 虽然它起作用，但它违反了建议：“优先使用基本类型而不是基本类型的包装类”。使用装箱基本类型的包装类进行批量操作的性能后果可能是致命的。



## 什么时候写自己的接口

现在你知道你应该通常使用标准的函数式接口来优先编写自己的接口。 但是，你应该什么时候写自己的接口？ 当然，**如果没有一个标准模块能够满足您的需求**，例如，如果需要一个带有三个参数的`Predicate`，或者一个抛出检查异常的`Predicate`，那么需要编写自己的代码。 但有时候你应该编写自己的函数式接口，即使与其中一个标准的函数式接口的结构相同。

考虑我们的老朋友`Comparator<T>`，它的结构与`ToIntBiFunction<T, T>`接口相同。 即使将前者添加到类库时后者的接口已经存在，使用它也是错误的。 `Comparator`值得拥有自己的接口有以下几个原因。 

1. 首先，它的名称每次在API中使用时都会提供优秀的文档，并且使用了很多。
2. 其次，`Comparator`接口对构成有效实例的构成有强大的要求，这些要求构成了它的普遍契约。 通过实现接口，就要承诺遵守契约。 
3. 最后，接口配备很多了有用的默认方法来转换和组合多个比较器。

如果需要一个函数式接口与`Comparator`共享以下一个或多个特性，应该认真考虑编写一个专用函数式接口，而不是使用标准函数式接口：

- 它将被广泛使用，并且可以从描述性名称中受益。
- 它拥有强大的契约。
- 它会受益于自定义的默认方法。

如果选择编写你自己的函数式接口，请记住它是一个接口，因此应非常小心地设计。

> 请注意，`EldestEntryRemovalFunction`接口标有`@FunctionalInterface`注解。 这种注解在类型类似于`@Override`。 这是一个程序员意图的陈述，它有三个目的：它告诉读者该类和它的文档，该接口是为了实现lambda表达式而设计的；它使你保持可靠，因为除非只有一个抽象方法，否则接口不会编译; 它可以防止维护人员在接口发生变化时不小心地将抽象方法添加到接口中。 **始终使用`@FunctionalInterface`注解标注你的函数式接口**。

最后一点应该是关于在 api 中使用函数接口的问题。不要提供具有多个重载的方法，这些重载在相同的参数位置上使用不同的函数式接口，如果这样做可能会在客户端中产生歧义。这不仅仅是一个理论问题。`ExecutorService`的`submit`方法可以采用`Callable<T>`或`Runnable`接口，并且可以编写需要强制类型转换以指示正确的重载的客户端程序避免此问题的最简单方法是不要编写在相同的参数位置中使用不同函数式接口的重载。要“明智地使用重载”。

总之，现在Java已经有了lambda表达式，因此必须考虑lambda表达式来设计你的API。 在输入上接受函数式接口类型并在输出中返回它们。 一般来说，最好使用`java.util.function.Function`中提供的标准接口，但请注意，在相对罕见的情况下，最好编写自己的函数式接口。

