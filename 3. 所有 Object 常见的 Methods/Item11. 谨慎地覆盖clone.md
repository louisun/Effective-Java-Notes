# Item11: 谨慎地覆盖 clone

`Cloneable` 接口的目的是作为对象的一个 **mixin 接口**，表明这样的对象允许克隆。不幸的是，它没有达到这个目的。它的主要缺点是缺少`clone`方法（这个接口只是标识作用），而`Object`的 `clone` 方法是 `protected` 的。即使是反射调用也可能失败，因为不能保证对象具有可访问的 `clone` 方法。



## 如何实现一个行为良好的 clone方法



果一个类实现了`Cloneable`接口，那么`Object`的`clone`方法将返回该对象的逐个属性（field-by-field）拷贝；否则会抛出`CloneNotSupportedException`异常。这是一个非常反常的接口使用，而不应该被效仿。 **通常情况下，实现一个接口用来表示可以为客户做什么。但对于`Cloneable`接口，它会修改父类上受保护方法的行为，**即 `public clone 方法 ` 



**它创建对象而不需要调用构造方法。**这个机制是脆弱的、危险的和不受语言影响的。

`clone` 方法的通用规范很薄弱的。 以下内容是从 `Object` 规范中的约定内容：

> 创建并返回此对象的副本。 “复制（copy）”的确切含义可能取决于对象的类。 一般意图是，对于任何对象 `x`，表达式`x.clone() != x`返回 true，并且`x.clone().getClass() == x.getClass()`也返回 true，但它们不是绝对的要求，但通常情况下，`x.clone().equals(x)`返回 true，当然这个要求也不是绝对的。拷贝对象会导致创建它的类的一个新实例，但它同时也会要求拷贝内部的数据结构。**这个过程没有调用构造器。**



根据约定，这个方法返回的对象应该通过调用`super.clone`方法获得的。 如果一个类和它的所有父类（Object除外）都遵守这个约定，情况就是如此，`x.clone().getClass() == x.getClass()`。

根据约定，返回的对象应该独立于被克隆的对象。 为了实现这种独立性，在返回对象之前，可能需要修改由super.clone返回的对象的一个或多个属性。



这种机制与构造方法链很相似，只是它没有被强制执行；如果一个类的`clone`方法返回一个通过调用构造方法获得而不是通过调用`super.clone`的实例，那么**编译器不会抱怨**，但是如果一个类的子类调用了`super.clone`，返回的对象包含错误的类，从而阻止子类 clone 方法正常执行。如果一个类重写的 clone 方法是有 final 修饰的，那么这个约定可以被安全地忽略，因为子类不需要担心。但是，**如果一个final类有一个不调用`super.clone`的`clone`方法，那么这个类没有理由实现`Cloneable`接口，因为它不依赖于`Object`的`clone`实现的行为。**

假设你希望在一个类中实现`Cloneable`接口，它的父类提供了一个行为良好的 `clone`方法。首先调用 `super.clone`。 得到的对象将是原始的完全功能的复制品。 在你的类中声明的任何属性将具有与原始属性相同的值。 如果每个属性包含原始值或对不可变对象的引用，则返回的对象可能正是你所需要的，在这种情况下，不需要进一步的处理。 例如，对于`PhoneNumber`类，情况就是这样，但是请注意，**不可变类永远不应该提供`clone`方法，因为这只会浪费复制。** 有了这个警告，以下是`PhoneNumber`类的clone方法：

```java
// Clone method for class with no references to mutable state
@Override 
public PhoneNumber clone() {
    try {
        return (PhoneNumber) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();  // Can't happen
    }
}
```

为了使这个方法起作用，`PhoneNumber`的类声明必须被修改，以表明它实现了`Cloneable`接口。 **虽然Object类的clone方法返回Object类，但是这个clone方法返回`PhoneNumber`类。** **这样做是合法和可取的，因为Java支持协变返回类型。 换句话说，重写方法的返回类型可以是重写方法的返回类型的子类。** 这消除了在客户端转换的需要。 在返回之前，我们必须将Object的super.clone的结果强制转换为`PhoneNumber`，但保证强制转换成功。

super.clone的调用包含在一个try-catch块中。 这是因为Object声明了它的clone方法来抛出`CloneNotSupportedException`异常，这是一个检查时异常。 由于`PhoneNumber`实现了Cloneable接口，所以我们知道调用super.clone会成功。 这里引用的需要表明`CloneNotSupportedException`应该是未被检查的。



## 浅拷贝的灾难：对象里引用了可变对象——要递归调用 clone

如果对象包含的域引用可变对象的属性，则前面显示的简单clone实现可能是灾难性的。



```java
public class Stack {

    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
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

    // Ensure space for at least one more element.
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```



假设你想让这个类可以克隆。 如果clone方法仅返回super.clone()调用的对象，那么生成的Stack实例在其size 属性中具有正确的值，**但elements属性引用与原始Stack实例相同的数组。** **修改原始实例将破坏克隆中的不变量**，反之亦然。 你会很快发现你的程序产生了无意义的结果，或者抛出`NullPointerException`异常。

调用Stack类中的唯一构造方法，这种情况永远不会发生。 实**际上，clone方法作为另一种构造方法; 必须确保它不会损坏原始对象，并且正确地创建被克隆对象中的约束条件。** 为了使Stack上的clone方法正常工作，它必须复制stack 对象的内部。 最简单的方法是对元素数组递归调用clone方法：

```java
// Clone method for class with references to mutable state
@Override public Stack clone() {
    try {
        Stack result = (Stack) super.clone();
        result.elements = elements.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

请注意，我们不必将elements.clone的结果转换为Object[]数组。**自 Java 1.5 开始， 在数组上调用clone会返回一个数组，其运行时和编译时类型与被克隆的数组相同。**  事实上，数组是clone 机制的唯一有力的用途。

**还要注意，如果elements属性是final的，则以前的解决方案将不起作用，因为克隆将被禁止向该属性分配新的值**。 这是一个基本的问题：**像序列化一样，Cloneable体系结构与引用可变对象的final 属性的正常使用不兼容，除非可变对象可以在对象和其克隆之间安全地共享。** **为了使一个类可以克隆，可能需要从一些属性中移除 final 修饰符。**



## 递归调用也不够的情况，要深拷贝



仅仅递归地调用`clone`方法并不总是足够的。 例如，假设您正在为哈希表编写一个`clone`方法，其内部包含一个哈希桶数组，每个哈希桶都指向“键-值”对链表的第一项。 为了提高性能，该类实现了自己的轻量级单链表，而没有使用`java`内部提供的`java.util.LinkedList`：



```java
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;
    private static class Entry {
        final Object key;
        Object value;
        Entry  next;

        Entry(Object key, Object value, Entry next) {
            this.key   = key;
            this.value = value;
            this.next  = next;  
        }
    }
    ... // Remainder omitted
}
```



假设你只是递归地克隆哈希桶数组，就像我们为`Stack`所做的那样：

```java
// Broken clone method - results in shared mutable state!
@Override public HashTable clone() {
    try {
        HashTable result = (HashTable) super.clone();
        result.buckets = buckets.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```



**虽然被克隆的对象有自己的哈希桶数组，但是这个数组引用与原始数组相同的链表，这很容易导致克隆对象和原始对象中的不确定性行为。** **要解决这个问题，你必须复制包含每个桶的链表。** 下面是一种常见的方法：

```java
// Recursive clone method for class with complex mutable state
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;

    private static class Entry {
        final Object key;
        Object value;
        Entry  next;

        Entry(Object key, Object value, Entry next) {
            this.key   = key;
            this.value = value;
            this.next  = next;  
        }

        // Recursively copy the linked list headed by this Entry
        Entry deepCopy() {
            return new Entry(key, value,
                next == null ? null : next.deepCopy());
        }
    }

    @Override public HashTable clone() {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];
            for (int i = 0; i < buckets.length; i++)
                if (buckets[i] != null)
                    result.buckets[i] = buckets[i].deepCopy();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
    ... // Remainder omitted
}
```



私有类HashTable.Entry已被扩充以支持“深度复制”方法。 HashTable上的clone方法分配一个合适大小的新哈希桶数组，迭代原来哈希桶数组，深度复制每个非空的哈希桶。 Entry上的`deepCopy`方法递归地调用它自己以复制由头节点开始的整个链表。 如果哈希桶不是太长，这种技术很聪明并且工作正常。**但是，克隆链表不是一个好方法，因为它为列表中的每个元素消耗一个栈帧（stack frame）。 如果列表很长，这很容易导致堆栈溢出。** 为了防止这种情况发生，**可以用迭代来替换`deepCopy`中的递归：**



```java
// Iteratively copy the linked list headed by this Entry
Entry deepCopy() {
   Entry result = new Entry(key, value, next);
   for (Entry p = result; p.next != null; p = p.next)
      p.next = new Entry(p.next.key, p.next.value, p.next.next);
   return result;
}
```





克隆复杂可变对象的最后一种方法是调用`super.clone`，**将结果对象中的所有属性设置为其初始状态，然后调用更高级别的方法来重新生成原始对象的状态。** 以HashTable为例，bucket属性将被初始化为一个新的bucket数组，并且 `put(key, value)`方法（未示出）被调用用于被克隆的哈希表中的键值映射。 这种方法通常产生一个简单，合理的优雅clone方法，其运行速度不如直接操纵克隆内部的方法快。 **虽然这种方法是干净的，但它与整个Cloneable体系结构是对立的，因为它会盲目地重写构成体系结构基础的逐个属性对象复制。**

**与构造方法一样，clone 方法绝对不可以在构建过程中，**调用一个可以重写的方法。如果 clone 方法调用一个在子类中重写的方法，则在子类有机会在克隆中修复它的状态之前执行该方法，很可能导致克隆和原始对象的损坏。因此，我们在前面讨论的 put(key, value)方法应该时 final 或 private 修饰的。（如果时 private 修饰，那么大概是一个非 final 公共方法的辅助方法）。

**Object 类的 clone方法被声明为抛出`CloneNotSupportedException`异常，但重写方法时不需要。 `public clone`方法应该省略`throws`子句，因为不抛出检查时异常的方法更容易使用。**

**在为继承设计一个类时，通常有两种选择，但无论选择哪一种，都不应该实现 Clonable 接口。**你可以选择通过实现正确运行的受保护的 clone方法来模仿Object的行为，该方法声明为抛出CloneNotSupportedException异常。 这给了子类实现Cloneable接口的自由，就像直接继承Object一样。 或者，可以选择不实现工作的 clone方法，并通过提供以下简并clone实现来阻止子类实现它：



```java
// clone method for extendable class not supporting Cloneable
@Override
protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
```

还有一个值得注意的细节。 如果你编写一个实现了`Cloneable`的线程安全的类，记得它的clone方法必须和其他方法一样需要正确的同步。  **Object 类的clone方法是不同步的，所以即使它的实现是令人满意的，也可能需要编写一个返回super.clone()的同步clone方法。**



## 总结



回顾一下，实现Cloneable的所有类应该重写 `public clone`方法，而**这个方法的返回类型是类本身**。 **这个方法应该首先调用**`super.clone`，**然后修复任何需要修复的属性。** 通常，这意味着复制任何包含内部“深层结构”的可变对象，并用指向新对象的引用来代替原来指向这些对象的引用。虽然这些内部拷贝通常可以通过递归调用clone来实现，但这并不总是最好的方法。 如果类只包含基本类型或对不可变对象的引用，那么很可能是没有属性需要修复的情况。 这个规则也有例外。 例如，表示序列号或其他唯一ID的属性即使是基本类型的或不可变的，也需要被修正。

这么复杂是否真的有必要？很少。 如果你继承一个已经实现了Cloneable接口的类，你别无选择，只能实现一个行为良好的`clone`方法。 否则，通常你最好提供另一种对象复制方法。 对象复制更好的方法是提供一个复制构造方法或复制工厂。 复制构造方法接受参数，其类型为包含此构造方法的类，例如，



```java
// Copy constructor
public Yum(Yum yum) { ... };
```



复制工厂类似于复制构造方法的静态工厂：

```java
// Copy factory
public static Yum newInstance(Yum yum) { ... };
```



复制构造方法及其静态工厂变体与`Cloneable/clone`相比有许多优点：它们不依赖风险很大的语言外的对象创建机制；不要求遵守那些不太明确的惯例；不会与 `final` 属性的正确使用相冲突; 不会抛出不必要的检查异常; 而且不需要类型转换。

此外，**复制构造方法或复制工厂可以接受类型为该类实现的接口的参数。** 例如，按照惯例，所有通用集合实现都提供了一个构造方法，其参数的类型为`Collection`或`Map`。 基**于接口的复制构造方法和复制工厂（更适当地称为转换构造方法和转换工厂）允许客户端选择复制的实现类型，而不是强制客户端接受原始实现类型。** 例如，假设你有一个HashSet，并且你想把它复制为一个TreeSet。 clone方法不能提供这种功能，但使用转换构造方法很容易：`new TreeSet<>(s)`。

考虑到与Cloneable接口相关的所有问题，新的接口不应该继承它，新的可扩展类不应该实现它。 虽然实现Cloneable接口对于final类没有什么危害，但应该将其视为性能优化的角度，仅在极少数情况下才是合理的。 通**常，复制功能最好由构造方法或工厂提供。 这个规则的一个明显的例外是数组，它最好用 clone方法复制。**