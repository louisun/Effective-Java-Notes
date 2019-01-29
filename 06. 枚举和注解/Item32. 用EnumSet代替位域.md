# Item32. 用EnumSet代替位域



## 什么是位域



如果一个枚举类型的元素主要用在集合中，一般就使用`int`枚举模式，**将 2 的不同倍数赋予每个常量**：

```java
// Bit field enumeration constants - OBSOLETE!
public class Text {
    public static final int STYLE_BOLD          = 1 << 0;  // 1
    public static final int STYLE_ITALIC        = 1 << 1;  // 2
    public static final int STYLE_UNDERLINE     = 1 << 2;  // 4
    public static final int STYLE_STRIKETHROUGH = 1 << 3;  // 8

    // Parameter is bitwise OR of zero or more STYLE_ constants
    public void applyStyles(int styles) { ... }
}
```

**这种表示法让你用OR位运算将几个常量合并到一个集合中，称作位域（bit field）。**



位域表示法也允许**利用位操作**，有效地执行像`union`（联合）和`intersection`（交集）这样的集合操作。但位域有着`int`枚举常量的所有缺点，甚至更多。当位域以数字形式打印时，翻译位域比翻译简单的`int`枚举常量要困难得多。甚至，要遍历位域表示的所有元素也没有很容易的方法。



## EnumSet



`EnumSet`就是一个集合 `Set`，里面可以存放 `Enum`



有些程序员**优先使用枚举**而非`int`常量，他们在**需要传递多组常量集时**，仍然倾向于使用位域。其实没有理由这么做，因为还有更好地替代方法。`java.util`包提供了`EnumSet`类来有效的表示**从单个枚举类型中提取的多个值得多个集合。**



这个类实现`Set`接口，提供了丰富的功能、类型安全性，以及可以从任何其他`Set`实现中得到的互用性。但是在内部具体的实现上，**每个`EnumSet`就是用单个`long`来表示，因此它的性能比得上位域的性能**。批处理，如`removeAll` 和 `retainAll`，**都是利用位算法来实现的，就像手工替位域实现的那样。但是可以避免手工位操作时容易出现的错误以及不大雅观的代码**，因为`EnumSet`替你完成了这项艰巨的工作。



```java
import java.util.EnumSet;
import java.util.Set;

public class Text {
	public enum Style {
		BOLD, ITALIC, UNDERLINE, STRIKETHROUGH
	}

	// 任意 Set 都可以传入，但 EnumSet 毫无疑问是最好的
	public void applyStyles(Set<Style> styles) { ...}


	public static void main(String[] args) {
		Text text = new Text();
        // EnumSet 提供了丰富的静态方法来创建即可，比如 of
		text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
	}
}

```



注意`applyStyles()`方法采用的是`Set<Style >` 而非`EnumSet <Style>`.虽然看起来好像所有的客户端都可以将`EnumSet`传到这个方法，但是最好还是接受接口类型而非接受实现类型。这是考虑到可能会有特殊的客户端要传递一些其他的`Set`实现，并且没有什么明显的缺点。



总之，**仅仅因为枚举类型将被用于集合中，所以没有理由用位属性来表示它**。 `EnumSet`类将位属性的简洁性和性能与条目30中所述的枚举类型的所有优点相结合。`EnumSet`的一个真正缺点是，它不像 Java 9 那样创建一个不可变的`EnumSet`，但是在即将发布的版本中可能会得到补救。 同时，你可以用`Collections.unmodifiableSet`封装一个`EnumSet`，但是简洁性和性能会受到影响。

