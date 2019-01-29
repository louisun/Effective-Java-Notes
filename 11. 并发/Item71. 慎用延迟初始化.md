# Item71. 慎用延迟初始化

**延迟初始化**（lazy initialization）是延迟到**需要域的值时才将它初始化**的这种行为。如果永远不需要这个值，这个域就永远不会被初始化。这种方法既适用于**静态域**，也适用于**实例域**。虽然延迟初始化主要是一种**优化**，但它也可以用来打破类和实例初始化中的有害循环。



就像大多数的优化一样，对于延迟初始化，最好建议“**除非绝对必要，否则就不要这么做**（见 Item 55）。**延迟初始化就像一把双刃剑。它降低了初始化类或者创建实例的开销，却增加了「访问被延迟初始化的域」的开销。**根据延迟初始化的域最终需要初始化的比例、初始化这些域要多少开销，以及每个域多久被访问一次，延迟初始化（就像其他的许多优化一样）实际上降低了性能。

也就是说，延迟初始化有它的好处。**如果域只在类的实例部分被访问，并且初始化这个域的开销很高，可能就值得进行延迟初始化。**要确定这一点，唯一的办法就是测量类在用和不用延迟初始化时的性能差别。

当有多个线程时，延迟初始化是**需要技巧**的。如果两个或者多个线程共享一个延迟初始化的域，采用某种形式的同步是很重要的，否则就可能造成严重的Bug。本条目中讨论的所有初始化方法都是线程安全的。



> 在大多数情况下，正常的初始化要优先于延迟初始化。



下面例子正常初始化实例域：

```java
private final FieldType field = computeFieldValue();
```



## 同步方法进行延迟初始化



如果用延迟优化来破坏初始化的循坏，就要使用「**同步方法**」，因为它是最简单、最清楚的替代方法：

```java
private FieldType field;

synchronized FieldType getField(){
    if(field == null){
        field = computeFieldValue();
    }
    return field;
}
```



这两种习惯模式（正常的初始化和使用了同步访问方法的延迟初始化）应用到静态域上时保持不变，除了给域和访问方法声明添加了 `static` 修饰符之外。



## 静态域的延迟初始化



如果处于性能考虑而需要对「**静态域**」使用延迟初始化，就使用 **lazy initialization holder class** 模式，这种模式保证了类要到被用到的时候才会被初始化：



```java
private static class FieldHolder{
	static final FieldType field = computeFieldValue();   
}

static Field getField(){
    retrun FieldHolder.field;
}
```



当 `getField` 第一次被调用的时，它第一次读取 `FieldHolder.field`，导致 `FieldHolder` 类得到初始化。**这种模式的魅力在于，`getField` 方法没有被同步，并且只执行一个域访问，因此延迟初始化实际上并没有增加任何访问成本。**现代的虚拟机在初始化该类的时候，同步域的访问。一旦这个类被初始化，虚拟机将修补代码，以便后续对该域的访问不会导致任何测试或同步。



## 实例域的延迟初始化



如果处于性能考虑，要对「**实例域**」进行延迟初始化，就应该使用「**双重检查模式**」。这种模式避免了在域被初始化之后访问这个域时的「**锁定开销**」（Item 67）。这种模式背后的思想是：两次检查域的值，第一次检查不锁定，看看域是否被初始化，第二次检查时有锁定。只有当第二次检查发现域没有被初始化，再回调用 `computeFieldValue` 方法对域进行参数。如果域已经被初始化就不会有锁定。域被声明为 `volatile` 非常重要（见 Item 66）。



```java
// 域被声明为 volatile
private volatile FieldType field;

FieldType getField(){
    FieldType result = field;
    if(result == null){
        synchronized(this){
            if(result == null){
                field = result = computeFieldValue();
            }
        }
    }
    return result;
}

// 没有局部变量 result 的情况
FieldType getField(){
    if(field == null){
        synchronized(this){
            if(field == null){
                field = computeFieldValue();
            }
        }
    }
    return field;
}
```



这个 `result` 变量的作用是确保 `field` 与只在已经被初始化的情况下读取一次。虽然不是严格需要，但是可以提升性能。





虽然对「静态域」也可以使用双重校验模式，但没有理由这么做，因为「**lazy initialization holder class idiom**」是更好的选择。



双重校验模式的两个变量值得一提，有时可能需要延迟初始化一个**可以接受重复初始化**的实例域。如果出现这种情况，就可以使用「双重校验」的一个变形：「**单重校验模式**」，即省去第二次检查。



```java
private volatile FieldType field;

FieldType getField(){
    FieldType result = field;
    if(result == null){
		field = result = computeFieldValue();
    }
    return result;
}
```







本条目讨论的所有初始化方法都适用于基本类型的域，以及对象的引用域。单、双重校验模式应用到数值类型的基本类型域时，就会用 0 来检验这个域（这是数值型变量的默认值），而不是 `null`。



如果你**不在意是否每个线程都重新计算域的值**，并且域的类型为**基本类型**，而不是 `long`或者 `double` 类型，就可以选择从单重检査模式的域声明中删除 `volatile` 修饰符。这种变体称之为 racy single-check idiom。它加快了某些架构上的域访问，代价是增加了**额外的初始化**（直到访问该域的每个线程都进行一次初始化）。这显然是一种特殊的方法，不适合于日常的使用。然而， `String` 实例却用它来缓存它们的散列码。



简而言之，大多数的域应该正常地进行初始化，而不是延迟初始化。如果为了达到性能目标，或者为了破坏有害的初始化循环，而必须延迟初始化一个域，就可以使用相应的延迟初始化方法。对于实例域，就使用双重检查模式（**double-check idiom**）；对于静态域，则使用**lazy initialization holder class idiom**。对于可以接受重复初始化的实例域，也可以考虑使用单重检查模式（**single-check idiom**）。

