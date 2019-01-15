# Item38. 检查参数的有效性



本章探讨方法设计的几个方面：如何处理参数、返回值、设计方法签名、编写文档。



检查参数有效性：应该在文档中清晰指明参数的限制。如果传递无效参数，应该很快失败，并清楚地出现适当的异常。如果方法没有检查参数，可能出现几种糟糕的情况：

1. 方法在处理过程中失败，产生令人费解的异常。
2. 方法可以正常返回，但计算得到错误的结果。
3. 最糟糕的是，方法可以正常返回，但却使某个对象处于被破坏的状态，将来可能在某个不想关的点上引发错误。

也就是说，不能正确地验证参数，可以导致违反**失败原子性** failure atomicity (v3-Item76）。



对于公有和受保护的方法，使用 JavaDoc 的`@throws`标签来记录当参数值的限制被违背时将抛出的异常。比较典型的异常就是 `IllegalArgumentException`、`IndexOutOfBoundsException`、`NullPointerException`。 一旦记录了对方法参数的限制，并且记录了违反这些限制时将引发的异常，那么强制执行这些限制就很简单了。

```java
/**
 * Returns a BigInteger whose value is (this mod m). This method
 * differs from the remainder method in that it always returns a
 * non-negative BigInteger.
 *
 * @param m the modulus, which must be positive
 * @return this mod m
 * @throws ArithmeticException if m is less than or equal to 0
 */

public BigInteger mod(BigInteger m) {
    if (m.signum() <= 0)
        throw new ArithmeticException("Modulus <= 0: " + m);
    ... // Do the computation
}
```



对非公有的方法，可以控制在哪些情况下被调用，通常使用「**断言 assertion**」来检查参数，断言失败会抛出 `AssertionError`，启用断言要对虚拟机传递 "-ea" 或者 "-enableassertions" 参数。



```java
// Private helper function for a recursive sort
private static void sort(long a[], int offset, int length) {
    assert a != null;
    assert offset >= 0 && offset <= a.length;
    assert length >= 0 && length <= a.length - offset;
    ... // Do the computation
}
```



对有些参数，方法本身没用到，被保存起来以后使用，这种情况检验参数尤为重要。比如构造器。



不过有一个例外，在方法执行计算前不去检查参数有效性，就是「检查参数的工作很昂贵」或者不切实际时。比如 `Collections.sort(List)`，列表中所有对象要是 `Comparable` 的，但没有必要去「**提前比较**」，因为 sort 方法本来就会对元素比较，如果不能比较则抛出 `ClassCastException`，提前检查参数没有意义。但其他情况其实都应该比较，否则会导致违反「失败原子性」。



有时候，**有些计算会「隐式」执行必要的有效性检查**，如果不成功会抛出错误，比如以后会讲的 「异常转译 exception translation」技术，讲计算过程抛出的异常转换为正确的异常。



不是说对参数的任何限制都是好事，设计方法时应使它们尽可能通用，复合实际需要。



**简而言之，每当编写方法或者构造器的时候，应该考虑它的参数有哪些限制。应该把这些限制写到文档中，并且在这个方法体的开头处，通过显式的检查来实施这些限制。**养成这样的习惯是非常重要的。只要有效性检查有一次失败，你为必要的有效性检査所付出的努力便都可以连本带利地得到偿还了。