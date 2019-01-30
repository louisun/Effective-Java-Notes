# Item75. 考虑自定义的序列化形式



**当你在时间紧迫的情况下设计一个类时，一般合理的做法是把工作重心集中在设计最佳的 API 上。**有时候，这意味着要发行一个「用完后即丢弃」的实现，因为你知道以后会在新版本中将它替换掉。正常情况下，这不成问题，但是，**如果这个类实现了 `Serializable` 接口，并且使用了默认的序列化形式，你就永远无法彻底摆脱那个应该丢弃的实现了。** 它将永远牵制住这个类的序列化形式。这不只是一个纯理论的问题，在Java 平台类库中已经有几个类出现了这样的问题，比如 `Biglnteger`。



如果没有先认真考虑「**默认的序列化形式是否合适**」，则**不要贸然接受**。接受默认的序列化形式是一个非常重要的决定，你需要「**从灵活性、性能和正确性**」多个角度对这种编码形式进行考察。一般来讲，**只有当你自行设计的自定义序列化形式与默认的序列化形式基本相同时**，才能接受默认的序列化形式。



考虑一个以对象为根的对象图，相对于它的「物理表示法」而言，该对象的默认序列化形式是一个比较有效的编码形式，换句话说，**默认的序列化形式应该只包含该对象锁表示的「逻辑数据」，而逻辑数据与物理表示法应该是各自独立的。**



## 逻辑数据与物理表示相同

如果一个对象的「物理表示法」等同于逻辑内容，可能就适用于「默认的序列化形式」。例如下面这些仅仅表示人名的类：



```java
// Good candidate for default serialized form 
public class Name implements Serializable {

    /** 
     * Last name. Must be non-null. 
     * @serial 
     */
	private final String lastName;

    /** 
     * First name. Must be non-null. 
     * @serial 
     */ 
	private final String firstName;
    
    /** 
     * Middle name, or null if there is none. 
     * @serial 
     */ 
	private final String middleName;

	... // Remainder omitted
}
```



从逻辑的角度看，一个名字包含 3 个字符串，分别表示姓、名、中间名。`Name` 中的实例域精确反应了其逻辑内容。



即使你确定了默认的序列化形式是合适的，**通常还必须提供一个 `readObject` 方法以保证约束关系和安全性。**对于`Name`这个类而言， `readObject`方法必须确保 `lastName`和 `firstName`是非`null`的。Item 76 和 Item 78 将详细地讨论这个问题。



注意，虽然 `lastName`、 `firstName` 和 `middleName` 域是私有的，但是它们仍然有**相应的注释文档**。这是因为，**这些私有城定义了一个公有的 API，即这个类的序列化形式**，并且该公有的 API 必须建立文档。`@serial` 标签告诉 Javadoc 工具，把这些文档信息放在有关序列化形式的特殊文档页中。



## 逻辑数据与物理表示不同



下面的例子与 `Name` 不同，是另一个极端，表示了一个「**字符串列表**」：



![](https://bucket-1255905387.cos.ap-shanghai.myqcloud.com/2019-01-30-13-25-57_r36.png)



从逻辑上将，这个类表示一个「**字符串序列**」，但从物理意义上，它是一个双向链表。默认序列化实际上会镜像出链表中的所有项，以及这些项直接的所有双向链接。



 当一个对象的物理表示法与逻辑数据内容有「**实质性区别**」时，使用「默认序列化」有以下 4 个缺点：

1. **它使这个类的导出 API 永远地束缚在该类的内部表示法上。**在上面的例子中， `StringList` 私有的`Entry`类变成了公有 API 的一部分。**如果在将来的版本中，内部表示法发生了变化， `StringList` 类仍将需要接受链表形式的输入，并产生链表形式的输出。** 这个类*永远也摆脱不掉维护链表项所需要的所有代码，即使它不再使用链表作为内部数据结构了也仍然需要这些代码。*
2. **它会消耗过多的空间。**在上面的例子中，序列化形式既表示了链表中的每个项，也表示了所有的链接关系，这是不必要的。**这些链表项以及链接只不过是实现细节，不值得记录在序列化形式中**。因为这样的**序列化形式过于庞大**，所以，把它写到磁盘中，或者网络上发送都将非常慢。
3. **它会消耗过多的时间。**「序列化逻辑」**并不了解对象图的拓扑关系，所以它必须要经过个昂贵的图遍历过程。**在上面的例子中，沿着 `next` 引用进行遍历是非常简单的
4. **它会引起栈溢出。**「默认的序列化过程」要对「对象图执行一次**递归遍历**」，即使**对于中等规模的对象图，这样的操作也可能会引起栈溢出**。在测试机器上，如果 `StringList` 实例包含 1258 个元素，对它进行序列化就会导致栈溢出。到底多少个元素会引发栈溢出，这要取决于 JVM 的具体实现以及Java启动时的命令行参数，（比如 Heap size 的`-Xms`与`-Xmx`的值）有些实现可能根本不存在这样的问题。



对于 `StringList` 类，**合理的序列化形式可以非常简单，只需先包含链表中字符串的数目，然后紧跟着这些字符串即可。**这样就构成了 `StringList` 所表示的逻辑数据，**与它的物理表示细节脱离**。下面是 StringList 的一个修订版本，它包含 `writeObject` 和 `readObject` 方法，用来实现这样的序列化形式。顺便提醒一下， **`transient` 修饰符表明这个实例域将从一个类的默认序列化形式中省略掉**：



```java
import java.io.*;

// StringList with a reasonable custom serialized formΩ
public final class StringList implements Serializable {
    private transient int size   = 0;
    private transient Entry head = null;

    // No longer Serializable!
    private static class Entry {
        String data;
        Entry  next;
        Entry  previous;
    }

    // Appends the specified string to the list
    public final void add(String s) {  }

    /**
     * Serialize this {@code StringList} instance.
     *
     * @serialData The size of the list (the number of strings
     * it contains) is emitted ({@code int}), followed by all of
     * its elements (each a {@code String}), in the proper
     * sequence.
     */
    private void writeObject(ObjectOutputStream s)
            throws IOException {
        s.defaultWriteObject();
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (Entry e = head; e != null; e = e.next)
            s.writeObject(e.data);
    }

    private void readObject(ObjectInputStream s)
            throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        int numElements = s.readInt();

        // Read in all elements and insert them in list
        for (int i = 0; i < numElements; i++)
            add((String) s.readObject());
    }

    // Remainder omitted
}
```



注意，尽管 `StringList` 的所有域都是瞬时的（transient），但 `writeObject` 方法的首要任务仍是调用 `defaultWriteObject`， `readObject` 方法的首要任务则是调用 `defaultReadObject`。如果所有的实例域都是瞬时的，从技术角度而言，不调用  `defaultWriteObject` 和 `defaultReadObject` 也是允许的，但是不推荐这样做。**即使所有的实例域都是 `transient` 的，调用  `defaultWriteObject` 也会影响该类的序列化形式，从而极大地增强灵活性**。这样得到的序列化形式允许在以后的发行版本中增加非 `transient` 的实例域，并且还能保持向前或者向后兼容性。如果某一个实例将在未来的版本中被序列化，然后在前一个版本中被反序列化，那么，后增加的域将被忽略掉。如果旧版本的 `readObject` 方法没有调用 `defaultReadObject`，反序列化过程将失败，引发 StreamCorrupted Exception 异常



注意，尽管 `writeObject` 方法是私有的，它也有文档注释。这与 `Name` 类中私有域的文档注释是同样的道理。**该私有方法定义了一个公有的 API，即序列化形式，并且这个公有的 API 应该建立文档。如**同域的 `@serial` 标签一样，方法的`@serialData`标签也告知 Javadoc工具，要把该文档信息放在有关序列化形式的文档页上。



套用以前对性能的讨论形式，如果平均字符串长度为 10 个字符， `StringList` 修订版本的序列化形式就只占用原序列化形式一半的空间。在测试机器上，同样是 10 个字符长度的情况下`StringList`修订版的序列化速度比原版本的快 2 倍。最终，修订版中不存在栈溢出的问题，因此，对于可被序列化的 `StringList`的大小也没有实际的上限。



虽然默认的序列化形式对于 `StringList` 类来说只是不适合而已，对于有些类，情况却变得更加糟糕。对于 `StringList`，默认的序列化形式不够灵活，并且执行效果不佳，但是序列化和反序列化 StringList 实例会产生对原始对象的忠实拷贝，它的约束关系没有被破坏，从这个意义上讲，这个序列化形式是正确的。但是，如果对象的约束关系要依赖于特定于实现的细节对于它们来说，情况就不是这样了。



## 对象的约束关系依赖于实现细节

例如，考虑散列表的情形。它的物理表示法是一系列包含“键一值（key-value）”项的散列桶。到底一个项将被放在哪个桶中，这是该键的散列码的一个函数，一般情况下，不同的JVM实现不保证会有同样的结果。实际上，即使在同一个JVM实现中，也无法保证每次运行都会一样。因此，对于散列表而言，接受默认的序列化形式将会构成一个严重的Bug。对散列表对象进行序列化和反序列化操作所产生的对象，其约束关系会遭到严重的破坏。



## 其他

无论你是否使用默认的序列化形式，当 `defaultWriteObject` 方法被调用的时候，每一个未被标记为 `transient` 的实例域都会被序列化。因此，每一个可以被标记为 `transient` 的实例域都应该做上这样的标记。这包括那些冗余的域，即它们的值**可以根据其他“基本数据域”计算而得到的域，比如缓存起来的散列值**。它也包括那些“其值依赖于 JVM 的某一次运行”的域，比如一个`long`域代表了一个指向本地数据结构的指针。**在决定将一个域做成非 `transient` 的之前，请一定要确信它的值将是该对象逻辑状态的一部分**。**如果你正在使用一种自定义的序列化形式，大多数实例域，或者所有的实例域则都应该被标记为 `transient`，就像上面例子中的 `StringList` 那样。**



如果你正在使用默认的序列化形式，并且把一个或者多个域标记为 `transient`，则要记住，当一个实例被反序列化的时候，这些域将被初始化为它们的默认值：对于对象引用域，默认值为 `null`；对于数值基本域，默认值为 `0`；对于 `boolean`域，默认值为 `false`。如果这些值不能被任何 `transient`域所接受，你就必须提供一个 `readObject`方法，它首先调用 `defaultReadObject`，然后把这些 `transient` 域恢复为可接受的值。另一种方法是，这些域可以被延迟到第一次被使用的时候才真正被初始化（见 Item 71）。



无论你是否使用默认的序列化形式，**如果在读取整个对象状态的任何其他方法上强制任何同步，则也必须在对象序列化上强制这种同步。**因此，如果你有一个线程安全的对象，它通过同步每个方法实现了它的线程安全，并且你选择使用默认的序列化形式，就要使用下列的 `writeObject` 方法：



```java
private synchronized void writeObject(ObjectOutputStream s) throws IOException{
    s.defaultWriteObject();
}
```





如果你把同步放在 `writeObject` 方法中，就必须确保它遵守与其他动作相同的锁排列约束条件，否则就有遭遇资源排列死锁的危险。



不管你选择了哪种序列化形式，都要为自己编写的每个可序列化的类声明一个显式的序列版本 UID。这样可以避免序列版本UID成为潜在的不兼容根源。而且，这样做也会带来小小的性能好处。如果没有提供显式的序列版本UID，就需要在运行时通过一个高开销的计算过程产生一个序列版本UID。



要声明一个序列版本UID非常简单，只要在你的类中增加下面一行：



```java
private static final long serialVersionUID = 随机Long值;
```





在编写新的类时，为随机 Long 值选择什么值并不重要。通过在该类上运行 serialver 工具，你就可以得到一个这样的值，但是，如果你凭空编造一个数值，那也是可以的。如果你想修改一个没有序列版本UID的现有的类，并希望新的版本能够接受现有的序列化实例，就必须使用那个自动为旧版本生成的值。如通过在旧版的类上运行 serialver工具，可以得到这个数值—被序列化的实例为之存在的那个数值。

如果你想为一个类生成一个新的版本，这个类与现有的类不兼容，那么你只需修改序列版本UID声明中的值即可。结果，前一版本的实例经序列化之后，再做反序列化时会引发`InvalidClassException`异常而失败。

总而言之，当你决定要将一个类做成可序列化的时候（见 Item 74），请仔细考虑应该采用什么样的序列化形式。只有当默认的序列化形式能够合理地描述对象的逻辑状态时，才能使用默认的序列化形式;否则就要设计一个自定义的序列化形式，通过它合理地描述对象的状态。你应该分配足够多的时间来设计类的序列化形式，就好像分配足够多的时间来设计它的导出方法一样。正如你无法在将来的版本中去掉导出方法一样，你也不能去掉序列化形式中的域;它们必须被永久地保留下去，以确保序列化兼容性。选择错误的序列化形式对于一个类的复杂性和性能都会有永久的负面影响。