# Item31. 用实例域代替序数



很多枚举天生与一个单独的 `int` 值相关联，所有枚举都有一个 `ordinal()` 方法，返回每个枚举常量在类型中的数字位置。

```java
// Abuse of ordinal to derive an associated value - DON'T DO THIS
public enum Ensemble {
    SOLO,   DUET,   TRIO, QUARTET, QUINTET,
    SEXTET, SEPTET, OCTET, NONET,  DECTET;

    public int numberOfMusicians() { return ordinal() + 1; }
}
```



不要用上面这样的枚举，如果将常量重新排序，`numberOfMusicians()`函数将遭到破坏。



## 自己写一个实力域，避免用 `ordinal()`

**永远不要根据枚举的序数到底出与它关联的值，而是将它保存在一个实例域中：**



```java
public enum Ensemble {
	SOLO(1), DUET(2), TRIO(3), QUARTET(4), QUINTET(5), SEXTET(6), SEPTET(7), OCTET(8),
    DOUBLE_QUARTET(8), NONET(9), DECTET(10), TRIPLE_QUARTET(12);

	private final int numberOfMusicians; // 实力域(序数)

	Ensemble(int size) {
		this.numberOfMusicians = size;
	}

	public int numberOfMusicians() {
		return numberOfMusicians;
	}
}

```





**`Enum` 规范中谈到 `ordinal()` 方法说道：大多数程序员都不需要这个方法。**  它被设计用于基于枚举的通用数据结构，如`EnumSet`和`EnumMap`。除非你在编写这样数据结构的代码，否则最好避免使用`ordinal`方法。