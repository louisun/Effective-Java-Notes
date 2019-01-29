# Item26. 优先考虑泛型



一般来说，将集合声明参数化，以及使用JDK提供的泛型类型和方法，这些都不会太困难。**编写自己的泛型会困难一些。**



考虑下面这个简单堆栈实现：

```java
// Object-based collection - a prime candidate for generics
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }

    public boolean isEmpty() {
        return size == 0;
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```



这个类是泛型化的主要备选对象，可以适当强化这个类来利用泛型。从实际情况看，必须转换从堆栈里弹出的对象，以及困难运行是啊比的那些转换。将类泛型化的第一个步骤是给它的声明添加一个或多个类型参数。这个例子中有一个类型参数，它表示堆栈的元素类型，这个参数名称通常为 `E`。



下一步是用相应的类型参数替换所有 `Object` 类型：



```java
// Initial attempt to generify Stack - won't compile!
public class Stack<E> {
    private E[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new E[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(E e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public E pop() {
        if (size == 0)
            throw new EmptyStackException();
        E result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }
    ... // no changes in isEmpty or ensureCapacity
}
```



通常，将得到一个错误或警告，这个类也不例外，幸运的是这个类只产生一个错误：

```java
Stack.java:8: generic array creation
        elements = new E[DEFAULT_INITIAL_CAPACITY];
```



如同上一个条目讲的，不能创建不可具体化类型数组，如 `E`。有两种解决方法，第一种是直接绕过创建泛型数组的禁令：创建一个 Object 数组，并将它转换为数组类型，但这样又会使编译器产生一条警告（合法但不是类型安全的）。



```java
Stack.java:8: warning: [unchecked] unchecked cast
found: Object[], required: E[]
        elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
```



编译器可能无法证明你的程序是类型安全的，但你可以证明，不加限制的类型强制转换不会损害程序的类型安全。 有问题的数组（元素）保存在一个私有属性中，永远不会返回给客户端或传递给任何其他方法。 保存在数组中的唯一元素是那些传递给`push`方法的元素，它们是`E`类型的，所以未经检查的强制转换不会造成任何伤害。



一旦证明未经检查的强制转换是安全的，请尽可能缩小范围。 在这种情况下，**构造方法只包含未经检查的数组创建，所以在整个构造方法中抑制警告是合适的**。 通过添加一个注解来执行此操作，Stack可以干净地编译，并且可以在没有显式强制转换或担心`ClassCastException`异常的情况下使用它：



```java
// The elements array will contain only E instances from push(E).
// This is sufficient to ensure type safety, but the runtime
// type of the array won't be E[]; it will always be Object[]!
@SuppressWarnings("unchecked")
public Stack() {
    elements = (E[]) new Object[DEFAULT_INITIAL_CAPACITY];
}
```



消除 Stack 中的泛型数组创建错误的第二种方法是将属性元素的类型从`E[]`更改为`Object[]`。 如果这样做，会得到一个不同的错误：



```java
Stack.java:19: incompatible types
found: Object, required: E
        E result = elements[--size];
                           ^
```

可以通过将从数组中取得的元素转换为`E`来将此错误更改为警告：



```java
Stack.java:19: warning: [unchecked] unchecked cast
found: Object, required: E
        E result = (E) elements[--size];
```



因为E是不可具体化的类型，编译器无法在运行时检查强制转换。 再一次，你可以很容易地向自己证明，不加限制的转换是安全的，所以可以适当地抑制警告。 根据条目 24 的建议，我们只要在包含未受检转换的任务上禁止警告，而不是在整个pop方法上就可以了：



```java
// Appropriate suppression of unchecked warning
public E pop() {
    if (size == 0)
        throw new EmptyStackException();
    // push requires elements to be of type E, so cast is correct
    @SuppressWarnings("unchecked") E result = (E) elements[--size];
    
    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```





两种消除泛型数组创建的技术都有其追随者。 第一个更可读：数组被声明为`E[]`类型，清楚地表明它只包含 `E` 实例。 它也更简洁：在一个典型的泛型类中，你从代码中的许多点读取数组; **第一种技术只需要一次转换（创建数组的地方），而第二种技术每次读取数组元素都需要单独转换。 因此，第一种技术是优选的并且在实践中更常用。** 但是，它确实会造成**堆污染**（heap pollution）：数组的运行时类型与编译时类型不匹配（除非`E`碰巧是Object）。 这使得一些程序员非常不安，他们选择了第二种技术，尽管在这种情况下堆的污染是无害的。



下面的程序演示了泛型`Stack`类的使用。 该程序以相反的顺序打印其命令行参数，并将其转换为大写。 对从堆栈弹出的元素调用String的`toUpperCase`方法不需要显式强制转换，并且确保自动生成的强制转换会成功：



```java
// Little program to exercise our generic Stack
public static void main(String[] args) {
    Stack<String> stack = new Stack<>();
    for (String arg : args)
        stack.push(arg);
    while (!stack.isEmpty())
        System.out.println(stack.pop().toUpperCase());
}
```



上面的例子似乎与条目 25 相矛盾，**条目 25 中鼓励列表优先于数组**。 实际上并不可能总是在泛型中使用列表。 Java不是生来就持列表，所以一些泛型类型（如`ArrayList`）必须在数组上实现。 其他的泛型类型，比如`HashMap`，是为了提高性能而实现的。

绝大多数泛型类型就像我们的Stack示例一样，它们的类型参数没有限制：可以创建一个`Stack<Object>`，`Stack<int[]>`，`Stack<List<String>>`或者其他任何对象的 Stack 引用类型。 请注意，不能创建基本类型的堆栈：尝试创建`Stack<int>`或`Stack<double>`将导致编译时错误。 这是Java泛型类型系统的一个基本限制。 可以使用基本类型的包装类来解决这个限制。



有一些泛型类型限制了它们类型参数的允许值。 例如，考虑 `java.util.concurrent.DelayQueue`，它的声明如下所示：



```java
class DelayQueue<E extends Delayed> implements BlockingQueue<E>
```



类型参数列表（`<E extends Delayed>`）要求实际的类型参数`E`是`java.util.concurrent.Delayed`的子类型。 这使得`DelayQueue`实现及其客户端可以利用`DelayQueue`元素上的`Delayed`方法，而不需要显式的转换或`ClassCastException`异常的风险。 类型参数 `E` 被称为限定类型参数。 请注意，子类型关系被定义为每个类型都是自己的子类型，因此创建`DelayQueue<Delayed>`是合法的。



## 总结



总之，使用泛型比使用需要在客户端代码中进行转换的类型来的更安全、容易。设计新类型的时候，要确保它们不需要这种转换就可以使用，这通常意味着要把类做成是泛型的。只要时间允许，就可以把现有的类型都泛型化。这对于这些类型的新用户来说会变得更轻松，又不会破怀现有客户端。



